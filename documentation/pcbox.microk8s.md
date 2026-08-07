# microk8s en pcbox

Instalación de un runtime mínimo de Kubernetes (`microk8s`) en el servidor
`pcbox`, vía Ansible, con el gate de CI que la protege antes de tocar
producción.

## 0. Por qué microk8s

`microk8s` es un runtime de Kubernetes de un solo binario (snap), pensado
para nodos únicos como `pcbox`. Se instala sin add-ons habilitados — nada de
`dns`, `ingress`, `hostpath-storage`, etc. — para mantener la superficie
mínima; si en el futuro un proyecto necesita un add-on puntual, se habilita
explícitamente en ese momento, no por defecto acá.

## 1. Instalación vía snap (confinamiento classic)

El playbook [`playbooks/installations/install-microk8s.yml`](../playbooks/installations/install-microk8s.yml)
instala `microk8s` usando el módulo `community.general.snap`, en modo
`classic` (microk8s lo requiere para acceder al sistema sin las
restricciones normales de sandboxing de snap):

```yaml
- name: Install microk8s via snap (classic confinement)
  community.general.snap:
    name: microk8s
    classic: true
    state: present
```

Este módulo es idempotente: si `microk8s` ya está instalado, correr el
playbook de nuevo no reporta `changed`.

## 2. Esperar a que microk8s esté listo

Instalar el snap no garantiza que el clúster ya esté operativo — hay que
esperar a que termine de arrancar. Como no existe un módulo de Ansible para
esto, se usa `ansible.builtin.command` con argv fijo (sin `shell:`, sin
interpolación de variables) y un timeout acotado:

```yaml
- name: Wait for microk8s to report ready
  ansible.builtin.command: microk8s status --wait-ready --timeout 180
  changed_when: false
```

`changed_when: false` es explícito porque esta tarea sólo consulta estado —
nunca modifica nada — así que nunca debe contarse como un cambio, ni siquiera
la primera vez.

## 3. Usuario SSH en el grupo `microk8s`

Para poder correr `microk8s kubectl` sin `sudo` (útil para diagnóstico manual
por SSH), el usuario que usa Ansible para conectarse (`jhon`, el valor del
secret `SSH_USER` — ver [`pcbox.bootstrap.md`](pcbox.bootstrap.md)) se agrega
al grupo `microk8s`:

```yaml
- name: Add SSH user to the microk8s group
  ansible.builtin.user:
    name: "{{ ansible_user }}"
    groups: microk8s
    append: true
```

`append: true` evita pisar otros grupos a los que el usuario ya pertenezca.

> **Nota sobre re-login:** el cambio de grupo en Linux sólo se aplica a
> sesiones nuevas — una sesión SSH que ya estaba abierta cuando corrió el
> playbook no lo ve hasta reconectar. Esto no afecta al propio playbook (las
> tareas de Ansible corren como root vía `become: true`, no dependen de la
> membresía de grupo), pero si un humano entra por SSH a correr
> `microk8s kubectl` a mano, y todavía tenía la sesión abierta desde antes de
> la primera instalación, tiene que cerrar sesión y volver a entrar para que
> el grupo nuevo tenga efecto.

## 4. Sin add-ons habilitados

El playbook no habilita ningún add-on de microk8s. Se puede verificar en
cualquier momento corriendo, en el servidor:

```bash
microk8s status
```

La sección de add-ons debe listarlos todos como deshabilitados.

## 5. Alcance solo por la tailnet

`microk8s` no expone ningún puerto a internet por sí mismo — no hay Ingress
ni add-on de exposición habilitado (ver punto 4). El único acceso al
servidor es el que ya existe desde el bootstrap
([`pcbox.bootstrap.md`](pcbox.bootstrap.md)): SSH sobre la IP de Tailscale
(`100.x.x.x`). Cualquier carga de trabajo que en el futuro se despliegue
sobre este clúster y necesite alcance externo deberá decidir explícitamente
cómo exponerse — eso queda fuera del alcance de este documento.

## 6. Cómo lo corre el gate de  CI

`.github/workflows/deploy.yml` corre en cada push a `master`
(y manualmente vía `workflow_dispatch`), en dos jobs:

1. **`lint`** (sin conexión al servidor, sin secrets de Tailscale/SSH):
   instala `ansible` + `ansible-lint` y corre `ansible-lint` (perfil
   default) sobre `playbooks/installations/install-microk8s.yml`. Si falla,
   el job de deploy no arranca.
2. **`ansible-deploy`** (depende de `lint` vía `needs: lint`): se conecta al
   servidor por Tailscale y clave SSH, y recién después llama a una
   composite action local en un único paso:
   [`.github/actions/ansible-deploy`](../.github/actions/ansible-deploy/action.yml)
   (`uses: ./.github/actions/ansible-deploy`, con `ssh-user`/`ssh-host`
   pasados por `with:` desde `secrets.SSH_USER`/`secrets.SSH_HOST` — una
   composite action no puede leer `secrets.*` directamente). Adentro de esa
   acción hay dos pasos, con nombres y comportamiento idénticos a como
   estaban antes en `deploy.yml`:

   1. `Dry-run de install-microk8s.yml (--check --diff)` — `continue-on-error: true`,
      porque la tarea que agrega el usuario al grupo `microk8s` falla en
      modo `--check` en un host donde microk8s todavía no está instalado, ya
      que el grupo lo crea el propio snap.
   2. `Ejecutar playbooks` — aplica los playbooks encadenados en una sola
      invocación de `ansible-playbook`: `remove-folder-test.yml` primero,
      seguido de `install-microk8s.yml`, para compartir el mismo contexto
      SSH y no duplicar el paso de conexión.

`deploy.yml` en sí mismo no lista playbooks — si se agrega uno nuevo que
también deba correr en cada deploy, se edita **solo**
`.github/actions/ansible-deploy/action.yml` (el paso "Ejecutar playbooks", y
su contraparte de dry-run si corresponde); `deploy.yml` no cambia.
