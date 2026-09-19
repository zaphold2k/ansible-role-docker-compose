# Ansible Role: docker-compose-manager

Installs and configures Docker Compose services on a server. This role dynamically creates `docker-compose.yaml` files from variable-based templates and manages their lifecycle using the `community.docker.docker_compose_v2` module.

## Features

- ✅ **Automatic validation**: Checks for Docker and Docker Compose v2 installation
- ✅ **Dynamic compose generation**: Creates compose files from Ansible variables
- ✅ **Lifecycle management**: Start, stop, restart, down, disable services
- ✅ **Image updates**: Pull latest images and recreate containers
- ✅ **Idempotent operations**: Safe to run multiple times
- ✅ **Smart recreation**: Only recreates containers when compose files change
- ✅ **Service gating**: Enable/disable services with the `enabled` flag
- ✅ **Networks & volumes**: Full support for custom networks and volumes
- ✅ **Project naming**: Optional custom project names

## Requirements

- Docker Engine installed on target hosts
- Docker Compose (installed as a Docker plugin)
- Ansible >= 2.12
- `community.docker` collection >= 3.0.0

## Installation

Install the required collection:

```bash
ansible-galaxy collection install community.docker
```

Or add to your `collections/requirements.yml`:

```yaml
collections:
  - name: community.docker
    version: ">=3.0.0"
```

## Role Variables

### Default Variables

Available in [defaults/main.yml](defaults/main.yml):

| Variable | Default | Description |
|----------|---------|-------------|
| `docker_compose_base_path` | `/opt` | Base directory for Docker Compose projects |
| `docker_services` | `{}` | Dictionary of services to manage |
| `docker_compose_file_mode` | `0640` | Mode of the generated compose file |
| `docker_compose_env_file_mode` | `0600` | Mode of the generated `.env` file |
| `docker_compose_files_owner` | `{{ ansible_user \| default("root") }}` | Owner of generated files |
| `docker_compose_files_group` | same as owner | Group of generated files |

### Service Definition Structure

Each service in `docker_services` supports the following structure:

```yaml
docker_services:
  <service_name>:
    enabled: true|false                    # Whether the service should be running
    compose_file: "path/to/compose.yaml"   # Path where compose file will be created
    project_name: "<name>"                 # Optional: compose project name (default: compose dir name)
    container_definition: []               # List of service definitions (see below)
    networks_definition: []                # Optional: list of network definitions
    volumes_definition: []                 # Optional: list of volume definitions
    env_vars: {}                           # Optional: KEY: value dict rendered to <compose dir>/.env
    env_file_template: "path/to/env.j2"    # Optional: custom Jinja2 template rendered to <compose dir>/.env
```

`project_name` is passed to every `docker_compose_v2` call (install, update, start,
stop, restart, down), so all phases always address the same compose project. When
omitted, Docker Compose derives it from the compose directory name.

#### Container Definition

The `container_definition` builds the `services` section of a Docker Compose file. Each item accepts any valid Docker Compose service properties:

```yaml
container_definition:
  - container_name: myapp
    image: "myapp:latest"
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    env_file:
      - path: .env
        required: true
    volumes:
      - ./data:/app/data
    networks:
      - mynetwork
    depends_on:
      - database
```

#### Network Definition

The `networks_definition` builds the `networks` section:

```yaml
networks_definition:
  - mynetwork:
      name: mynetwork
      driver: bridge
```

#### Volume Definition

The `volumes_definition` builds the `volumes` section:

```yaml
volumes_definition:
  - mydata:
      name: mydata
      driver: local
```

#### `.env` files (secrets)

The role can generate the `.env` file next to the compose file. The intended
pattern is: **keys live in the inventory, values live in an Ansible Vault**.

Option A — `env_vars` dict, rendered with the role's `templates/env.j2`:

```yaml
docker_services:
  cloudflared:
    env_vars:
      TUNNEL_TOKEN: "{{ vault_cloudflared_tunnel_token }}"
```

Option B — `env_file_template`, a custom Jinja2 file you own (useful for long
`.env` files with comments, or `.dist`-style files):

```yaml
docker_services:
  cloudflared:
    env_file_template: "{{ inventory_dir }}/templates/cloudflared.env.j2"
```

```ini
# templates/cloudflared.env.j2
TUNNEL_TOKEN={{ vault_cloudflared_tunnel_token }}
```

In both cases the values are expected to come from an encrypted vars file, e.g.
`group_vars/all/vault.yaml`:

```yaml
vault_cloudflared_tunnel_token: "eyJh..."
```

Notes:

- The `.env` is written with mode `0600` (`docker_compose_env_file_mode`) and the
  task runs with `diff: false`, so secrets never show up in `--diff` output.
- A change in the rendered `.env` counts as a definition change: the service is
  stopped and recreated, exactly as when the compose file changes.
- If neither `env_vars` nor `env_file_template` is set, the role does not touch
  `.env`; whatever exists on the host is left as is.
- Remember to reference the file from the container: `env_file: [{path: .env, required: true}]`.

## Usage

### Basic Usage

```yaml
- hosts: docker_hosts
  become: yes
  roles:
    - role: docker-compose-manager
```

### Complete Example

```yaml
- hosts: homeservers
  become: yes

  vars:
    docker_services:
      # Service with volumes
      nginx:
        enabled: true
        compose_file: "{{ docker_compose_base_path }}/nginx/docker-compose.yaml"
        container_definition:
          - container_name: nginx
            image: "nginx:alpine"
            restart: unless-stopped
            ports:
              - "80:80"
              - "443:443"
            volumes:
              - nginx_conf:/etc/nginx
              - nginx_data:/usr/share/nginx/html
            networks:
              - web
        networks_definition:
          - web:
              name: web
              driver: bridge
        volumes_definition:
          - nginx_conf:
              name: nginx_conf
          - nginx_data:
              name: nginx_data

      # Disabled service
      test_app:
        enabled: false
        compose_file: "{{ docker_compose_base_path }}/test/docker-compose.yaml"
        container_definition:
          - container_name: test
            image: "alpine:latest"

  roles:
    - role: docker-compose-manager
```

## Tags

The role supports granular control via Ansible tags:

| Tag | Action | Default run | Description |
|-----|--------|-------------|-------------|
| `install` | Install/Deploy | yes | Renders compose + `.env`, ensures enabled services are up. Recreates only when the definition changed. |
| `start` | Start | no (`never`) | `up -d` for enabled services |
| `stop` | Stop | no (`never`) | Stops every loaded service, regardless of `enabled` |
| `restart` | Restart | no (`never`) | `stop` + `start` |
| `down` | Teardown | no (`never`) | Stops and removes containers and networks of the project |
| `pull` | Pull | no (`never`) | Pulls images only |
| `update` | Update | no (`never`) | Pulls images and recreates containers when a new image was pulled |

Tags marked `never` only run when explicitly requested with `--tags`. To take a
service out of service set `enabled: false` **and** run `--tags down` (or `stop`);
the flag alone only prevents the role from starting it.

### Tag Examples

```bash
# Install and start all services
ansible-playbook docker-deploy.yml --tags install

# Update all services (pull images and restart)
ansible-playbook docker-deploy.yml --tags update

# Stop all services
ansible-playbook docker-deploy.yml --tags stop

# Restart specific host
ansible-playbook docker-deploy.yml --tags restart --limit myhost

# Complete teardown
ansible-playbook docker-deploy.yml --tags down
```

## How It Works

### 1. Validation Phase (`always` tag)
- Verifies Docker is installed
- Verifies Docker Compose v2 is available
- Validates that `docker_services` is defined

### 2. Installation Phase (`install` tag)
- Creates directory structure for each service
- Generates `docker-compose.yaml` from templates
- Generates `.env` when `env_vars` or `env_file_template` is defined
- If compose or `.env` changed: stops the service, then `up` with `pull: always` and `recreate: always`
- If nothing changed: `up` with `pull: missing` and `recreate: auto` (idempotent, brings up stopped services)

### 3. Update Phase (`update`/`pull` tags)
- Pulls latest images using `docker_compose_v2_pull`
- Recreates containers with updated images
- Only affects enabled services

### 4. Management Phase (`start`/`stop`/`restart`/`down` tags)
- Controls service state using `docker_compose_v2`
- Respects the `enabled` flag
- Idempotent operations

## Generated Compose File Example

Given the CloudFlared configuration above, this role generates:

```yaml
services:
  cloudflared:
    command: tunnel --no-autoupdate run
    container_name: cloudflared
    env_file:
      - path: .env
        required: true
    image: cloudflare/cloudflared:latest
    networks:
      - cloudflared
    restart: unless-stopped

networks:
  cloudflared:
    name: cloudflared
```

## Advanced Features

### Smart Recreation

The role automatically detects when compose files change and:
1. Stops the existing service
2. Recreates containers with new configuration
3. Starts the service again

### Conditional Execution

Use the `enabled` flag to control service state:

```yaml
docker_services:
  production_app:
    enabled: "{{ environment == 'production' }}"
    # ... rest of config
```

### Debug Mode

Use verbosity level 2 to see detailed debug information:

```bash
ansible-playbook docker-deploy.yml -vv
```

## Dependencies

- **Collection**: `community.docker` (version >= 3.0.0)
- **Modules used**:
  - `community.docker.docker_compose_v2`
  - `community.docker.docker_compose_v2_pull`


## TODO

- [ ] Add health check verification after service start
- [ ] Backup functionality before updates
- [ ] Rollback capability on failed updates