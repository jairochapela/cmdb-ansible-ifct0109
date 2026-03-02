# Guía: Rol `nginx_apparmor` y Playbook `web.yml`

## Descripción general

Esta guía documenta paso a paso la creación del rol Ansible **`nginx_apparmor`** y el playbook **`web.yml`**. El objetivo es desplegar nginx con un virtualhost personalizado y protegerlo mediante un perfil AppArmor en modo `enforce` o `complain`.

---

## Estructura del proyecto

```
CMDB/
├── inventarios/
│   └── produccion/
│       ├── hosts
│       └── group_vars/
│       └── host_vars/
├── roles/
│   └── nginx_apparmor/
│       ├── defaults/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       └── templates/
│           ├── apparmor_nginx.j2
│           └── site.conf.j2
└── web.yml
```

---

## Paso 1 — Crear el inventario

Archivo: `inventarios/produccion/hosts`

Define el grupo `servidores` con la IP del host objetivo:

```ini
[servidores]
192.168.201.200
```

---

## Paso 2 — Crear la estructura del rol

Desde la raíz del proyecto, genera los directorios del rol manualmente o con `ansible-galaxy`:

```bash
mkdir -p roles/nginx_apparmor/{defaults,handlers,tasks,templates}
```

O con ansible-galaxy:

```bash
ansible-galaxy init roles/nginx_apparmor
```

---

## Paso 3 — Definir las variables por defecto

Archivo: `roles/nginx_apparmor/defaults/main.yml`

Estas variables pueden sobreescribirse desde el inventario o desde el playbook:

```yaml
nginx_site_name: secure_site
nginx_port: 8000

nginx_root_dir: "/var/www/{{ nginx_site_name }}"
nginx_log_dir: "/var/log/nginx"

apparmor_mode: enforce   # enforce | complain
```

| Variable | Descripción | Valor por defecto |
|---|---|---|
| `nginx_site_name` | Nombre del virtualhost | `secure_site` |
| `nginx_port` | Puerto de escucha | `8000` |
| `nginx_root_dir` | Ruta raíz del sitio web | `/var/www/secure_site` |
| `nginx_log_dir` | Directorio de logs de nginx | `/var/log/nginx` |
| `apparmor_mode` | Modo AppArmor (`enforce` o `complain`) | `enforce` |

---

## Paso 4 — Crear las tareas del rol

Archivo: `roles/nginx_apparmor/tasks/main.yml`

Las tareas se ejecutan en el siguiente orden:

### 4.1 Instalar paquetes

```yaml
- name: Instalar nginx y apparmor
  apt:
    name:
      - nginx
      - apparmor
      - apparmor-utils
    state: present
    update_cache: yes
```

### 4.2 Crear el directorio raíz del sitio

```yaml
- name: Crear directorio del sitio
  file:
    path: "{{ nginx_root_dir }}"
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'
```

### 4.3 Crear la página de prueba

```yaml
- name: Crear página de prueba
  copy:
    dest: "{{ nginx_root_dir }}/index.html"
    content: |
      <html>
        <h1>Servidor protegido con AppArmor</h1>
      </html>
    owner: www-data
    group: www-data
    mode: '0644'
```

### 4.4 Configurar el virtualhost desde plantilla

```yaml
- name: Configurar virtualhost nginx
  template:
    src: site.conf.j2
    dest: "/etc/nginx/sites-available/{{ nginx_site_name }}"
  notify: Restart nginx
```

### 4.5 Desactivar el sitio por defecto de nginx

```yaml
- name: Desactivar default site
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: Restart nginx
```

### 4.6 Preparar permisos del directorio de logs

```yaml
- name: Asegurar permisos correctos en logs
  file:
    path: "{{ nginx_log_dir }}"
    state: directory
    owner: root
    group: adm
    mode: '0775'

- name: Añadir www-data al grupo adm
  user:
    name: www-data
    groups: adm
    append: yes
```

### 4.7 Crear archivos de log

Se crean los cuatro archivos de log necesarios (globales y por sitio):

```yaml
- name: Crear archivos de log si no existen
  file:
    path: "{{ nginx_log_dir }}/error.log"
    state: touch
    owner: www-data
    group: adm
    mode: '0640'

- name: Crear archivo access.log
  file:
    path: "{{ nginx_log_dir }}/access.log"
    state: touch
    owner: www-data
    group: adm
    mode: '0640'

- name: Crear archivos de log si no existen
  file:
    path: "{{ nginx_log_dir }}/{{ nginx_site_name }}.error.log"
    state: touch
    owner: www-data
    group: adm
    mode: '0640'

- name: Crear archivo access.log
  file:
    path: "{{ nginx_log_dir }}/{{ nginx_site_name }}.access.log"
    state: touch
    owner: www-data
    group: adm
    mode: '0640'
```

### 4.8 Activar el sitio (symlink)

```yaml
- name: Activar sitio
  file:
    src: "/etc/nginx/sites-available/{{ nginx_site_name }}"
    dest: "/etc/nginx/sites-enabled/{{ nginx_site_name }}"
    state: link
  notify: Restart nginx
```

### 4.9 Instalar el perfil AppArmor

```yaml
- name: Instalar perfil AppArmor básico para nginx
  template:
    src: apparmor_nginx.j2
    dest: /etc/apparmor.d/usr.sbin.nginx
  notify:
    - Reload AppArmor
    - Set AppArmor mode
    - Restart nginx
```

---

## Paso 5 — Crear los handlers

Archivo: `roles/nginx_apparmor/handlers/main.yml`

Los handlers se invocan mediante `notify` y se ejecutan al final del play si fueron notificados:

```yaml
- name: Reload AppArmor
  command: apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx

- name: Set AppArmor mode
  command: "aa-{{ apparmor_mode }} usr.sbin.nginx"

- name: Restart nginx
  systemd:
    name: nginx
    state: restarted
```

> **Nota:** El handler `Set AppArmor mode` usa la variable `apparmor_mode` para ejecutar `aa-enforce` o `aa-complain` según corresponda.

---

## Paso 6 — Crear las plantillas Jinja2

### 6.1 Plantilla del virtualhost nginx

Archivo: `roles/nginx_apparmor/templates/site.conf.j2`

```nginx
server {
    listen {{ nginx_port }};
    server_name localhost;

    root {{ nginx_root_dir }};
    index index.html;

    access_log {{ nginx_log_dir }}/{{ nginx_site_name }}.access.log;
    error_log {{ nginx_log_dir }}/{{ nginx_site_name }}.error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 6.2 Plantilla del perfil AppArmor

Archivo: `roles/nginx_apparmor/templates/apparmor_nginx.j2`

El perfil restringe qué recursos puede acceder el proceso `/usr/sbin/nginx`:

```
#include <tunables/global>

profile usr.sbin.nginx /usr/sbin/nginx {

  # Permisos básicos del sistema
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # Configuración nginx
  /etc/nginx/** r,

  /run/nginx.pid rw,

  # Directorio web
  {{ nginx_root_dir }}/ r,
  {{ nginx_root_dir }}/** r,

  # Logs
  {{ nginx_log_dir }}/ r,
  {{ nginx_log_dir }}/** rw,

  # Permitir red TCP
  network inet stream,
  network inet6 stream,

  # Denegaciones explícitas formativas
  deny /home/** rw,
  deny /root/** rw,
  deny /etc/shadow r,
}
```

| Regla | Descripción |
|---|---|
| `/etc/nginx/** r` | Lectura de la configuración de nginx |
| `/run/nginx.pid rw` | Lectura/escritura del PID |
| `nginx_root_dir/** r` | Lectura del contenido web |
| `nginx_log_dir/** rw` | Escritura de logs |
| `network inet stream` | Conexiones TCP IPv4 e IPv6 |
| `deny /home/** rw` | Bloqueo explícito al home de usuarios |
| `deny /etc/shadow r` | Bloqueo al fichero de contraseñas |

---

## Paso 7 — Crear el playbook `web.yml`

Archivo: `web.yml` (en la raíz del proyecto)

```yaml
---
  - name: "Web"
    hosts: servidores
    become: yes
    roles:
      - nginx_apparmor
```

- `hosts: servidores` — apunta al grupo definido en el inventario.
- `become: yes` — ejecuta las tareas con privilegios de superusuario.
- `roles` — aplica el rol `nginx_apparmor` al play.

---

## Paso 8 — Ejecutar el playbook

```bash
ansible-playbook -J -i inventarios/produccion/hosts web.yml
```

Para verificar el resultado en modo check (sin aplicar cambios):

```bash
ansible-playbook -J -i inventarios/produccion/hosts web.yml --check
```

Para cambiar el modo AppArmor en tiempo de ejecución:

```bash
ansible-playbook -J -i inventarios/produccion/hosts web.yml -e "apparmor_mode=complain"
```

> Nota: La opción `-J` sirve para hacer que Ansible solicite la contraseña de cifrado del *vault*. Si no se utilizan archivos de variables cifrados no es necesario indicar esta opción.

---

## Resultado esperado

Al finalizar la ejecución correctamente:

- nginx estará instalado y activo en el puerto `8000`.
- El sitio `secure_site` servirá la página de prueba desde `/var/www/secure_site`.
- El perfil AppArmor `/etc/apparmor.d/usr.sbin.nginx` estará cargado y activo en modo `enforce`.
- Los directorios y ficheros de log tendrán los permisos correctos para `www-data`.
