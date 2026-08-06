# Convenciones de `playbooks/`

Notas de contexto para cualquiera (humano o Claude) que agregue o modifique
archivos dentro de [`playbooks/`](../playbooks/). No es una skill ni un
comando — es referencia de proyecto.

## Qué es cada archivo

Cada archivo `.yml` dentro de `playbooks/` es un **playbook de Ansible**:
describe un conjunto de tareas a aplicar sobre el servidor `pcbox` (definido
en [`inventory/hosts.yml`](../inventory/hosts.yml)). Un playbook = una
responsabilidad concreta (instalar algo, configurar algo, eliminar algo) — no
un "playbook maestro" que hace de todo.

Ejemplo real, [`playbooks/add-configuration/create-folder-test.yml`](../playbooks/add-configuration/create-folder-test.yml):

```yaml
- name: Create folder-test on the server
  hosts: pcbox-server-prod
  become: true
  gather_facts: false

  tasks:
    - name: Create /home/jhon/folder-test
      ansible.builtin.file:
        path: /home/jhon/folder-test
        state: directory
        mode: '0755'
        owner: jhon
        group: jhon
```

## Estructura esperada de un playbook nuevo

- **`name:`** en ingles, describe la intención (ej. "Instalar microk8s", no
  "Playbook 3").
- **`hosts: pcbox-server-prod`** — apuntar siempre al host específico definido
  en `inventory/hosts.yml`, no a `all`. Hoy es el único servidor que hay, pero
  `all` corre el playbook contra cualquier host que se agregue al inventario
  en el futuro, aunque no sea el destino pensado para ese playbook.
- **`become: true`** cuando la tarea necesita privilegios de root.
- **`gather_facts:`** en `false` salvo que el playbook realmente necesite
  facts del host (mantiene los runs rápidos).
- **`tasks:`** usando módulos declarativos de Ansible
  (`ansible.builtin.*`, `community.general.*`, etc.), no `shell:`/`command:`
  sueltos salvo que no exista módulo equivalente — los módulos son los que
  garantizan idempotencia real.

## Reglas no negociables

- **Idempotente**: correr el playbook dos veces seguidas no debe fallar ni
  cambiar el resultado la segunda vez. El deploy (`.github/workflows/deploy.yml`)
  corre en *cada* push a `master`, no solo cuando cambia ese playbook.
- **Cero secretos**: nada de IPs reales, contraseñas, tokens o claves
  hardcodeadas en el YAML. Los valores reales van como secrets de GitHub
  Actions o placeholders en `inventory/hosts.yml`.
- **Solo Tailscale**: si un playbook expone un servicio, nunca a internet
  directo — solo alcanzable por la tailnet existente.
- **Documentado**: todo playbook nuevo que automatice un paso que antes se
  hacía a mano se acompaña de su doc correspondiente en
  [`documentation/`](../documentation/), con el mismo nivel de detalle que
  [`documentation/pcbox.bootstrap.md`](../documentation/pcbox.bootstrap.md).

## Naming

`verbo-objetivo.yml` en ingles, minúsculas, guiones (ej. `create-folder.yml`,
`install-microk8s.yml`) — igual que el resto del repo.

## Subcarpetas de `playbooks/`

Cada playbook nuevo va en la subcarpeta que corresponda según qué hace, no
suelto en la raíz de `playbooks/`:

- **`add-configuration/`**: agrega o ajusta algo que ya existe en el
  servidor — crear archivos/carpetas, agregar permisos, variables,
  configuración de un servicio ya instalado. No instala ni desinstala
  software.
- **`remove-configuration/`**: lo inverso de `add-configuration/` — borra
  archivos/carpetas, quita permisos o configuración que ya no hace falta. No
  desinstala software (eso es `uninstalls/`).
- **`installations/`**: instala software o paquetes nuevos (ej. microk8s,
  Docker). Después de correrlo, algo que no estaba ahora está.
- **`uninstalls/`**: remueve software o recursos del servidor — lo inverso de
  `installations/`. Después de correrlo, algo que estaba ya no está.

Si un playbook no encaja claramente en ninguna, va en `add-configuration/` por
defecto.

## Cómo se ejecutan

Vía `.github/workflows/deploy.yml`, que corre:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/<archivo>.yml -u "$SSH_USER" \
  --private-key ~/.ssh/deploy_key -e "ansible_host=$SSH_HOST"
```

en cada push a `master`. Hoy el workflow apunta a un único playbook
(`add-configuration/create-folder-test.yml`) — si se agrega uno nuevo que
también deba correr en cada deploy, hay que sumarlo (o reemplazarlo) ahí
explícitamente.

## Pedir aprobación antes de crear o modificar un playbook

No se crea ni se modifica ningún archivo de `playbooks/` sin aprobación
explícita previa. Antes de tocar código, explicar en el chat:

- **Qué** archivo se va a crear o modificar (ej. `playbooks/installations/install-microk8s.yml`).
- **Qué tareas** va a tener (o qué tareas cambian, si ya existe).
- **Por qué** esa forma de resolverlo — qué regla de la sección "Reglas no
  negociables" de arriba cumple, y qué alternativa se descartó (si hubo una).

y esperar confirmación del usuario antes de escribir el archivo. Es solo el paso de "explicar y preguntar" antes de proceder, en
cada creación o modificación.

