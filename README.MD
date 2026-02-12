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
| `docker_services` | `[]` | Dictionary of services to manage |

### Service Definition Structure

Each service in `docker_services` supports the following structure:

```yaml
docker_services:
  <service_name>:
    enabled: true|false                    # Whether the service should be running
    compose_file: "path/to/compose.yaml"   # Path where compose file will be created
    container_definition: []               # List of service definitions (see below)
    networks_definition: []                # Optional: list of network definitions
    volumes_definition: []                 # Optional: list of volume definitions
```

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

| Tag | Action | Description |
|-----|--------|-------------|
| `install` | Install/Deploy | Creates compose files and starts enabled services |
| `start` | Start | Starts all enabled services |
| `stop` | Stop | Stops all services (preserves containers) |
| `restart` | Restart | Restarts all enabled services |
| `down` | Teardown | Stops and removes containers, networks, volumes |
| `disable` | Disable | Stops services where `enabled: false` |
| `update` / `pull` | Update | Pulls latest images and recreates containers |

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

# Disable services marked as enabled: false
ansible-playbook docker-deploy.yml --tags disable

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
- Automatically stops services if compose file changed
- Starts enabled services with `pull: always`
- Recreates containers when configuration changes

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

- [ ] Manage `.env` files using Ansible Vault for secure credential handling
- [ ] Add health check verification after service start
- [ ] Backup functionality before updates
- [ ] Rollback capability on failed updates