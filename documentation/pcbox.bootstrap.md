# Bootstrap del servidor pcbox

Configuración inicial que se hizo a mano, una sola vez, en el servidor `pcbox` para dejarlo listo y que GitHub Actions pueda desplegar sobre él vía Ansible.

## 1. Instalar OpenSSH en el Ubuntu Server

Para poder conectarse por SSH a este servidor. Se hace conectándose directamente a la compu (teclado/monitor) o por la red local:

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

Para autenticar al servidor en la red de Tailscale (así el runner de GitHub Actions lo puede alcanzar sin exponer el servidor a internet).

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

## 3. Configurar una llave privada y pública para conectarse por SSH sin contraseña

GitHub Actions no puede tipear una contraseña, así que necesita conectarse por clave SSH, no por password. Si hoy se entra con contraseña, hacer esto una sola vez:

```bash
ssh-keygen -t ed25519 -f ./deploy_key -N ""
ssh-copy-id -i deploy_key.pub usuario@IP_DEL_SERVIDOR
```

`usuario` es el usuario del servidor con el que nos conectamos por SSH (el mismo que se guardó en el paso 4 con permisos de sudo sin contraseña) — este es el valor que va a ir en el secret `SSH_USER` más adelante. `IP_DEL_SERVIDOR` es la IP obtenida en el paso 1 (o ya la de Tailscale del paso 2, si el servidor ya está en la tailnet).

Este comando genera dos archivos: `deploy_key` (la clave **privada**) y `deploy_key.pub` (la clave **pública**, que es la que se copia al servidor). La clave privada, `deploy_key`, es la que va a usar GitHub Actions para autenticarse por SSH sin contraseña — este archivo **no se sube al repo** (está en `.gitignore`), su contenido se pega directo en el secret `SSH_PRIVATE_KEY` más adelante.

Probar que funciona sin pedir password:

```bash
ssh -i deploy_key usuario@IP_DEL_SERVIDOR
```

Si ya existe una clave que se usa para conectarse, se puede saltar directo al paso 1.

Para ver la clave privada que va a usar GitHub Actions para conectarse al servidor por SSH sin contraseña (sino por clave privada), y así cargarla en el secret `SSH_PRIVATE_KEY`:

```bash
cat deploy_key
```

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