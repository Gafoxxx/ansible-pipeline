# ansible-pipeline

Configura con **Ansible** las VMs creadas por [terraform_for_each_vm](https://github.com/Gafoxxx/terraform_for_each_vm):

| Host | Qué se instala | Puerto |
|---|---|---|
| `nginx` | Nginx sirviendo la app [Teclado](https://github.com/Gafoxxx/Teclado) | 80 |
| `jenkins` | Docker + Docker Compose con Jenkins LTS, SonarQube 8.2 y PostgreSQL 12 | 80 (Jenkins), 9000 (SonarQube) |

## Estructura

```
.
├── ansible.cfg              # inventario, llave SSH, sin host key checking
├── inventory.ini.example    # plantilla (inventory.ini se genera, no se versiona)
├── playbook.yml             # plays para nginx y jenkins
├── docker-compose.yml       # jenkins + sonarqube + db
├── jenkins/
│   ├── Dockerfile           # Jenkins LTS + python, node, sshpass + plugins
│   ├── plugins.txt          # plugins a instalar en la imagen
│   └── init.groovy.d/
│       └── basic-security.groovy.override   # crea el usuario admin
└── secrets/                 # generado localmente, no versionado
    └── jenkins_admin_password
```

## Qué hace el playbook

**Play `nginx`**
1. Instala `nginx` y `git`, y deja el servicio habilitado.
2. Borra la página por defecto y clona Teclado (rama `main`) en `/var/www/html`.
3. Deja `/var/www/html` a nombre de `ansible_user`, para que se pueda redesplegar sin sudo.

**Play `jenkins`**
1. Agrega el repositorio oficial de Docker e instala `docker-ce` y `docker-compose-plugin`.
2. Agrega el usuario al grupo `docker` y fija `vm.max_map_count=262144`, que SonarQube (Elasticsearch) necesita.
3. Copia `docker-compose.yml`, la carpeta `jenkins/` y un `.env` con las credenciales del admin (modo `0600`).
4. Ejecuta `docker compose up -d --build`:
   - Construye la imagen `jenkinsci-local:lts`.
   - Levanta `jenkins`, `sonarqube` y `db`.

## Uso

Requisitos: Ansible ≥ 2.15, las VMs ya creadas y la llave `~/.ssh/id_rsa` autorizada en ellas (Terraform la configura).

1. Crear `inventory.ini` a partir de la plantilla con las IPs de `terraform output servers`:

   ```ini
   [nginx]
   nginx-vm ansible_host=<IP_NGINX>

   [jenkins]
   jenkins-vm ansible_host=<IP_JENKINS>

   [all:vars]
   ansible_user=adminuser
   ```

2. Ejecutar:

   ```bash
   ansible all -m ping
   ansible-playbook playbook.yml
   ansible-playbook playbook.yml --limit jenkins   # sólo una VM
   ```

### Despliegue completo en un comando

Con los tres repos clonados en la misma carpeta, el script `deploy.sh` de esa carpeta hace todo el flujo:
- corre `terraform apply`;
- genera `inventory.ini` a partir de los outputs;
- espera a que haya SSH;
- corre este playbook.

```bash
./deploy.sh           # crea / actualiza todo
./deploy.sh destroy   # elimina la infraestructura
```

## Acceso

| Servicio | URL | Credenciales |
|---|---|---|
| Teclado | `http://<IP_NGINX>` | — |
| Jenkins | `http://<IP_JENKINS>` | usuario `admin`, contraseña en `secrets/jenkins_admin_password` |
| SonarQube | `http://<IP_JENKINS>:9000` | `admin` / `admin` (cambiarla al primer ingreso) |

La contraseña de Jenkins se genera la primera vez que corre el playbook, con el lookup `password` de Ansible, y después se reutiliza. Para cambiarla, edita ese archivo y vuelve a correr el playbook; el script de init actualiza la contraseña en cada arranque.

## Cambios realizados

Frente a la versión original de este repo:

- **Docker:** `apt_key` y Compose v1 (binario 1.29.2) se reemplazaron por `deb822_repository` con `signed_by` y el plugin `docker compose`.
- **Tareas que fallaban:**
  - Se quitó `docker exec -it`, porque Ansible no tiene TTY.
  - Se quitó `become_method: su`, porque pedía la contraseña del usuario.
- **PostgreSQL fijado en `postgres:12`.** SonarQube 8.2 no soporta versiones más nuevas.
- **Jenkins:**
  - **Imagen:** `madmilodz/jenkinsci:v3` traía Jenkins 2.387.2, y los plugins `latest` de `plugins.txt` ya exigen Jenkins ≥ 2.479, así que Jenkins arrancaba sin plugins. Se reemplazó por una imagen equivalente sobre `jenkins/jenkins:lts-jdk21` con python3, nodejs/npm y sshpass. Los plugins se instalan durante el build.
  - **Usuario admin:** se creó con `init.groovy.d`. Antes Jenkins quedaba abierto a internet sin login, porque `runSetupWizard=false`. Ahora se exige login para todo (`FullControlOnceLoggedInAuthorizationStrategy`, sin lectura anónima).
- **Nginx:** ahora despliega Teclado.
- **Archivos nuevos:** `ansible.cfg`, `inventory.ini.example` y `.gitignore` (`inventory.ini`, `secrets/`).
