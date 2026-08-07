# Bootstrap del servidor pcbox

Configuración inicial que se hizo a mano, una sola vez, en el servidor `pcbox` para dejarlo listo y que GitHub Actions pueda desplegar sobre él vía Ansible.

## 0. Instalación de Ubuntu Server

Configuración elegida durante el instalador de Ubuntu Server:

| Opción | Valor |
|---|---|
| Lenguaje | Español |
| Teclado | Inglés EEUU (Internacional con teclas muertas) |
| Tipo de instalación | Por defecto |
| Comunicación con otros equipos | Type eth |
| Proxy address | (vacío) |
| Disco | Kingston |
| Nombre | Jhonny Berdeja |
| Nombre del server | pcbox |
| Nombre de usuario | jhon |
| Contraseña | (ver nota abajo) |

Este usuario, `jhon`, es el que se usa en todos los pasos siguientes para conectarse por SSH — es el valor que va en el secret `SSH_USER`:

```
SSH_USER=jhon
```

> **Nota sobre la contraseña:** hay que guardarla para usarla en el paso 3 (copiar la clave pública con `ssh-copy-id`, que pide la contraseña una única vez) y en el paso 4 (entrar por SSH con contraseña antes de tener la clave configurada). Por seguridad, **no se guarda en texto plano en este documento ni en el repo** — guardarla en un gestor de contraseñas. Una vez configurada la clave SSH (paso 3) y el `sudo` sin contraseña (paso 4), esta contraseña deja de ser necesaria para el flujo automatizado.

## 1. Instalar OpenSSH en el Ubuntu Server

Como el servidor todavía no tiene SSH instalado, no se puede entrar por red — hay que conectarle un teclado y un monitor directamente, y conectarlo a internet (cable de red o WiFi, según lo que tenga disponible) para poder descargar el paquete.

Ya con acceso directo a la terminal del servidor:

```bash
sudo apt update
sudo apt install openssh-server -y
```

Verificar que el servicio esté corriendo:

```bash
sudo systemctl status ssh
```

Si no está activo:

```bash
sudo systemctl enable --now ssh
```

Obtener la IP local del servidor (la que se usa para conectarse por la red local, antes de tener Tailscale configurado):

```bash
ip addr show
```

o más simple:

```bash
hostname -I
```

Esta IP se usa para conectarse por SSH en los pasos siguientes (por ejemplo, para el `ssh-copy-id` del paso 3). **Guardarla**, la vamos a necesitar más adelante.

## 2. Instalar y configurar Tailscale

Con OpenSSH ya instalado (paso 1) y la IP local guardada, ya no hace falta teclado ni monitor — de acá en adelante se trabaja conectándose por SSH desde una PC cliente:

```bash
ssh jhon@IP_LOCAL_DEL_SERVIDOR
```

(pide la contraseña configurada en el paso 0). Ya conectado al servidor por SSH, instalar y configurar Tailscale, para autenticar al servidor en la red de Tailscale (así el runner de GitHub Actions lo puede alcanzar sin exponer el servidor a internet):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Luego iniciar sesión y conectar el equipo a la red Tailscale:

```bash
sudo tailscale up
```

Esto da un link para autenticarse con una cuenta (Google, Microsoft, GitHub, etc.). Se abre desde cualquier navegador (puede ser desde el celular) y se hace login ahí.

Una vez conectado el servidor a la tailnet, hay que obtener su IP de Tailscale (`100.x.x.x`). Para eso, conectarse a la tailnet desde una PC cliente (la propia compu, ya con Tailscale instalado y logueado con la misma cuenta) y correr:

```bash
tailscale status
```

Ahí va a aparecer el servidor `pcbox` listado junto a su IP `100.x.x.x`. También se puede ver desde la [consola web de Tailscale](https://login.tailscale.com/admin/machines). **Guardar esta IP**, es la que se va a usar como valor del secret `SSH_HOST` en GitHub Actions más adelante.

Para probar que la conexión funciona ahora por Tailscale (en vez de por la red local), desde la misma PC cliente:

```bash
ssh jhon@IP_TAILSCALE
```

Si conecta usando esa IP, Tailscale quedó bien configurado en el servidor.

## 3. Configurar una llave privada y pública para conectarse por SSH sin contraseña

Esta configuración se hace conectándose por SSH ya sobre la red de Tailscale, usando la IP `100.x.x.x` obtenida en el paso 2 (no hace falta estar en la misma red local que el servidor).

GitHub Actions no puede tipear una contraseña, así que necesita conectarse por clave SSH, no por password. Si hoy se entra con contraseña, hacer esto una sola vez, desde la PC cliente:

```bash
ssh-keygen -t ed25519 -f ./deploy_key -N ""
ssh-copy-id -i deploy_key.pub jhon@IP_TAILSCALE
```

(Si ya existe una clave que se usa para conectarse al servidor, se puede saltar el `ssh-keygen` y pasar directo al `ssh-copy-id` con esa clave.)

`jhon` es el usuario del servidor (paso 0) — es el valor del secret `SSH_USER`. `IP_TAILSCALE` es la IP obtenida en el paso 2 — es el valor del secret `SSH_HOST`.

Este comando genera dos archivos: `deploy_key` (la clave **privada**) y `deploy_key.pub` (la clave **pública**, que es la que se copia al servidor). La clave privada, `deploy_key`, es la que va a usar GitHub Actions para autenticarse por SSH sin contraseña — este archivo **no se sube al repo** (está en `.gitignore`).

Probar que la conexión funciona con la clave privada, sin que pida contraseña:

```bash
ssh -i deploy_key jhon@IP_TAILSCALE
```

Si conecta sin pedir password, la clave quedó bien configurada.

Para ver el contenido de la clave privada y poder usarla desde otro cliente — en este caso, para pegarla en el secret `SSH_PRIVATE_KEY` de GitHub Actions:

```bash
cat deploy_key
```

Copiar toda la salida, incluyendo las líneas `-----BEGIN...-----` y `-----END...-----`, y pegarla completa como valor del secret.

## 4. Configurar el usuario para que no tenga que escribir la contraseña de sudo

Ya que GitHub Actions no puede escribir la contraseña de forma interactiva.

Entrar al servidor con un usuario que tenga privilegios de admin (puede ser el mismo, si todavía deja loguearse y pedir la clave a mano):

```bash
ssh usuario@IP_TAILSCALE
```

Editar los sudoers de forma segura (mejor crear un archivo aparte en vez de tocar `/etc/sudoers` directo):

```bash
sudo visudo -f /etc/sudoers.d/github-deploy
```

Agregar esta línea (reemplazando `usuario` por el valor real del secret `SSH_USER`):

```
usuario ALL=(ALL) NOPASSWD:ALL
```

Guardar y salir. `visudo` valida la sintaxis automáticamente antes de guardar, así que si hay un error de tipeo avisa y no rompe nada.

Verificar sin salir de la sesión SSH (por si el paso anterior tuvo un error, no se pierde el acceso):

```bash
sudo -n true && echo "OK, no pide password"
```

## 5. Instalar microk8s

A diferencia de los pasos anteriores, esto se instala a mano y se mantiene fuera de Ansible/CI a propósito (no hay ningún playbook que lo instale ni lo gestione). Conectado por SSH sobre Tailscale:

```bash
sudo snap install microk8s --classic --channel=1.31/stable
```

Agregar al usuario `jhon` al grupo `microk8s` para no necesitar `sudo` en los comandos de `kubectl`:

```bash
sudo usermod -aG microk8s $USER
```

Esto no toma efecto en la sesión SSH actual hasta refrescar el grupo:

```bash
newgrp microk8s
```

(o cerrar y volver a entrar por SSH — cualquiera de las dos sirve).

Esperar a que el cluster esté listo (tarda un rato la primera vez, está bajando imágenes de containerd):

```bash
microk8s status --wait-ready
```

Verificar:

```bash
microk8s kubectl get nodes
```

Debería aparecer el nodo `pcbox` en estado `Ready`.

## 6. Extender el certificado del API server para Tailscale y preparar el kubeconfig para CI

Por defecto, el certificado que el API server de microk8s le muestra a quien se conecta solo es válido para la IP local del servidor — no para la IP de Tailscale (`100.x.x.x`, la misma ya guardada como secret `SSH_HOST` en el paso 2). Como el runner de GitHub Actions se conecta por Tailscale, hay que extender ese certificado para que también sea válido desde esa IP.

Editar la plantilla del certificado:

```bash
sudo nano /var/snap/microk8s/current/certs/csr.conf.template
```

En la sección `[alt_names]` va a haber algo como:

```ini
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = 127.0.0.1
IP.2 = 10.152.183.1
IP.3 = 192.168.x.x   ← la IP local del servidor
```

**Sin reemplazar ninguna línea existente**, agregar una entrada nueva con el próximo número disponible (si la última es `IP.3`, la nueva es `IP.4`):

```ini
IP.4 = 100.x.x.x
```

con la IP de Tailscale del servidor. Guardar y salir, y regenerar el certificado del API server a partir de la plantilla actualizada (sin reiniciar todo el cluster):

```bash
sudo microk8s refresh-certs -e server.crt
```

> **Nota:** el flag es `-e`/`--cert`, no `-c` (`-c`/`--check` solo revisa vencimientos, no regenera nada — da un error de "Path does not exist" si se confunde).

Esperar a que se estabilice y verificar que la IP de Tailscale quedó en el certificado:

```bash
microk8s status --wait-ready
openssl x509 -in /var/snap/microk8s/current/certs/server.crt -noout -text | grep -A5 "Subject Alternative Name"
```

Generar el kubeconfig y editar el campo `server:` para que apunte a la IP de Tailscale en vez de a la IP local (la CA no cambia, el resto del archivo queda igual):

```bash
microk8s config > ~/pcbox-kubeconfig.yaml
nano ~/pcbox-kubeconfig.yaml
```

```yaml
server: https://100.x.x.x:16443   # IP de Tailscale, no la local
```

Sacar el archivo del servidor a la PC cliente (contiene una clave privada — es un secreto, no se commitea al repo):

```bash
scp jhon@IP_TAILSCALE:~/pcbox-kubeconfig.yaml .
```

## 7. Datos que quedan de este proceso

| Dato | Qué es | De qué paso salió | Para qué es |
|---|---|---|---|
| `SSH_USER` | El usuario del servidor, `jhon` | Paso 0 (usuario creado durante la instalación de Ubuntu Server) | Usuario con el que GitHub Actions se conecta por SSH al servidor |
| `SSH_HOST` | La IP de Tailscale del servidor (`100.x.x.x`) | Paso 2 (`tailscale status` desde la PC cliente) | Host al que se conecta el runner de CI por SSH; es la misma IP que se usa en `server:` del kubeconfig (paso 6) |
| `SSH_PRIVATE_KEY` | La clave privada `deploy_key` generada con `ssh-keygen` | Paso 3 | Autenticación SSH del runner sin contraseña |
| `pcbox-kubeconfig.yaml` | El kubeconfig de microk8s, con `server:` editado para apuntar a la IP de Tailscale en vez de a la IP local | Paso 6 (`microk8s config` + edición manual) | Credencial para administrar el cluster de forma remota — vive solo en la PC cliente, pendiente de decidir cómo se le entrega a CI cuando haga falta desplegar |