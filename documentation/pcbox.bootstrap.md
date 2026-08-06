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

## 3. Configurar una llave privada y pública para conectarse por SSH sin contraseña

GitHub Actions no puede tipear una contraseña, así que necesita conectarse por clave SSH, no por password. Si hoy se entra con contraseña, hacer esto una sola vez:

```bash
ssh-keygen -t ed25519 -f ./deploy_key -N ""
ssh-copy-id -i deploy_key.pub usuario@IP_DEL_SERVIDOR
```

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