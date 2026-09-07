# Ansible Role: Rolling Deploy

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

An Ansible role for implementing zero-downtime rolling deployments across a cluster of nodes. This role enables controlled, sequential node updates by temporarily blocking upstream load balancer connections using iptables, ensuring high availability during deployments.

## Features

- 🔄 **Zero-downtime deployments** - Maintain service availability during updates
- 🎯 **Blue/green deployment support** - Seamless traffic switching between node sets
- 🔒 **Iptables-based traffic control** - Block/unblock load balancer connections
- 🐳 **Docker and non-Docker compatible** - Works with both deployment types
- 🌐 **IPv4 and IPv6 support** - Dual-stack network compatibility
- ⚡ **Sequential deployment control** - Configurable serial execution

## Requirements

- **Ansible**: 2.9 or higher
- **OS**: Linux distribution with iptables support (Ubuntu, Debian, CentOS/RHEL)
- **Privileges**: Root/sudo access for iptables management
- **Network**: Upstream load balancers configured to respect connection blocks

## Installation

Install from Ansible Galaxy:

```bash
ansible-galaxy install fccn.ansible_rolling_deploy
```

Or add to `requirements.yml` (pattern used by [fccn/nau_playbooks](https://github.com/fccn/nau_playbooks/blob/master/requirements.yml)):

```yaml
roles:
  # Rolling deploy role that uses iptables to manage load balancer connections
  # to a cluster of nodes applying a rolling update strategy
  - src: git+https://github.com/fccn/ansible-rolling-deploy.git
    version: main  # pin to a commit SHA in production
```

Then install with:

```bash
ansible-galaxy install -r requirements.yml -p vendor/roles
```

## Role Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `rolling_deploy_parent_servers_ipv4` | List of IPv4 addresses or hostnames of load balancers | `['172.24.1.81', '172.24.1.82']` |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `rolling_deploy_starting` | `false` | Set to `true` to block connections, `false` to unblock |
| `rolling_deploy_parent_servers_ipv6` | `[]` | List of IPv6 addresses of load balancers |
| `rolling_deploy_iptables_state` | Auto-calculated | Iptables rule state (`present` or `absent`) |
| `rolling_deploy_chains` | `[FORWARD, INPUT]` | Iptables chains to modify |
| `rolling_deploy_iptables_jump` | `DROP` | Iptables target used to block parent servers. `DROP` silently discards every packet, new and already-established alike |
| `rolling_deploy_graceful_reject_seconds` | `0` | Optional grace period, in seconds. When `> 0`, blocks only *new* connections with REJECT first, waits this long, then falls back to `rolling_deploy_iptables_jump` for everything. `0` (default) disables this and preserves the original immediate-block behavior |
| `rolling_deploy_graceful_reject_jump` | `REJECT` | Target used only during the graceful phase above |
| `rolling_deploy_graceful_reject_with` | `tcp-reset` | Reject type used only during the graceful phase above. If set to `tcp-reset`, the role automatically restricts that rule to `-p tcp` since the kernel rejects `tcp-reset` on a rule that doesn't match TCP; other reject types (e.g. `icmp-port-unreachable`) apply to all protocols as usual |

See [defaults/main.yml](defaults/main.yml) for complete variable definitions.

## Configuration Examples

### Define Load Balancers Explicitly

```yaml
rolling_deploy_parent_servers_ipv4:
  - 172.24.1.81
  - 172.24.1.82
  - myloadbalancer.priv.fccn.pt
```

### Use Inventory Groups

Define load balancers in your inventory:

```ini
[load_balancers]
lb01-dev.myservice.fccn.pt ansible_host=172.24.1.81
lb02-dev.myservice.fccn.pt ansible_host=172.24.1.82
```

Reference in playbook:

```yaml
rolling_deploy_parent_servers_ipv4: "{{ groups['load_balancers'] | map('extract', hostvars, ['ansible_host']) | list }}"
```

### IPv6 Support

```yaml
rolling_deploy_parent_servers_ipv6:
  - 2001:db8::1
  - 2001:db8::2
```

## Usage

### Basic Pattern

1. **Block traffic** - Stop new connections to the node
2. **Deploy application** - Update services/containers
3. **Unblock traffic** - Re-enable load balancer connections

### Complete Playbook Example

```yaml
- name: Rolling deployment for application servers
  hosts: app_servers
  serial: "{{ serial_number | default(1) }}"  # Deploy one node at a time
  become: true
  gather_facts: true
  
  vars:
    rolling_deploy_parent_servers_ipv4:
      - 172.24.1.81
      - 172.24.1.82

  tasks:
    - name: Block load balancer connections
      import_role:
        name: fccn.ansible_rolling_deploy
      vars:
        rolling_deploy_starting: true
      when: groups['app_servers'] | length > 1

    - name: Wait for active connections to drain
      wait_for:
        timeout: 30

    # Your deployment tasks here
    - name: Deploy application
      docker_container:
        name: myapp
        image: myapp:latest
        state: started
        restart: true

    - name: Verify application health
      uri:
        url: http://localhost:8080/health
        status_code: 200
      retries: 10
      delay: 3

    - name: Re-enable load balancer connections
      import_role:
        name: fccn.ansible_rolling_deploy
      vars:
        rolling_deploy_starting: false
      when: groups['app_servers'] | length > 1
```

### Minimal Example

```yaml
- name: Simple rolling deployment
  hosts: web_servers
  serial: 1
  become: true
  tasks:
    - name: Start rolling deploy
      import_role:
        name: fccn.ansible_rolling_deploy
      vars:
        rolling_deploy_starting: true
        rolling_deploy_parent_servers_ipv4: ['172.24.1.81']

    # ... your deployment tasks ...

    - name: Finish rolling deploy
      import_role:
        name: fccn.ansible_rolling_deploy
      vars:
        rolling_deploy_starting: false
        rolling_deploy_parent_servers_ipv4: ['172.24.1.81']
```

### Reusable `close_node` / `open_node` Pattern

For repositories that deploy many services, the recommended pattern (as used in [fccn/nau_playbooks](https://github.com/fccn/nau_playbooks/tree/master/tasks)) is to centralise the rolling-deploy + keepalived + health-check logic in two reusable task files and import them from every service playbook.

**`tasks/close_node.yml`** — drain traffic before deploy:

```yaml
- name: Lower keepalived priority to force VIP swap
  import_role:
    name: ansible-keepalived
  vars:
    keepalived_priority_override: 1
  when: keepalived_vrrp_instances is defined and (keepalived_vrrp_instances | length > 0)

- name: Flush handlers so any pending restarts run before we block traffic
  meta: flush_handlers

- name: Block load balancer connections
  import_role:
    name: ansible-rolling-deploy
  vars:
    rolling_deploy_starting: true
  when: rolling_deploy_enabled | default(true) | bool
```

**`tasks/open_node.yml`** — restore traffic after deploy:

```yaml
- name: Open load balancer connections
  import_role:
    name: ansible-rolling-deploy
  vars:
    rolling_deploy_starting: false
  when: rolling_deploy_enabled | default(true) | bool

- name: Restore keepalived priority
  import_role:
    name: ansible-keepalived
  vars:
    keepalived_priority_override: ""
  when: keepalived_vrrp_instances is defined and (keepalived_vrrp_instances | length > 0)
```

**`group_vars/all/rolling_deploy.yml`** — resolve load balancers once from inventory:

```yaml
---
rolling_deploy_parent_servers_ipv4: "{{ groups['balancer_servers'] | map('extract', hostvars, ['ansible_host']) | list }}"
```

**Service playbook usage** — import the helpers around the deploy task:

```yaml
- name: Deploy financial manager servers
  hosts: financial_manager_docker_servers
  serial: "{{ serial_number | default(1) }}"
  become: true
  gather_facts: true
  tasks:
    - import_tasks: tasks/close_node.yml
      when: (groups['financial_manager_docker_servers'] | length) > 1

    - name: Deploy app
      import_role:
        name: financial_manager_docker_deploy

    - import_tasks: tasks/healthcheck.yml

    - import_tasks: tasks/open_node.yml
      when: (groups['financial_manager_docker_servers'] | length) > 1
```

### Generic Rolling Execute Playbook

A common companion is a `rolling_execute.yml` playbook that runs an arbitrary command against a cluster with the same close/health/open cycle — useful for `docker pull`, rolling restarts, or one-off maintenance:

```yaml
---
- hosts: all
  serial: "{{ serial_number | default(1) }}"
  become: true
  gather_facts: true
  vars:
    rolling_deploy_enabled: true
  tasks:
    - assert:
        that: command is defined
        fail_msg: You need to pass -e command='...'

    - import_tasks: tasks/close_node.yml

    - name: Run command
      shell: "{{ command }}"
      register: exec_output

    - import_tasks: tasks/healthcheck.yml
    - import_tasks: tasks/open_node.yml
```

Invoke as:

```bash
ansible-playbook -i hosts.ini rolling_execute.yml \
  --limit mongo_docker_servers \
  -e "command='docker pull mongo:6.0'"
```

## How It Works

1. **Traffic Blocking Phase** (`rolling_deploy_starting: true`)
   - Inserts iptables rules at the top of FORWARD and INPUT chains
   - Blocks incoming connections from specified load balancers using `rolling_deploy_iptables_jump` (`DROP` by default) — this matches every packet from the load balancer, including ones belonging to already-established connections, so in-flight requests are cut immediately
   - If `rolling_deploy_graceful_reject_seconds` is set, the block is graceful instead: only *new* connection attempts are rejected (fast `tcp-reset`, so the load balancer notices and stops routing new traffic quickly) while already-established connections are left alone to finish naturally; after the configured number of seconds the role falls back to the standard `rolling_deploy_iptables_jump` rule covering all connection states, and only then returns control to the calling playbook

2. **Deployment Phase**
   - Node is isolated from new traffic
   - Safe to restart services or deploy updates
   - Health checks can be performed in isolation

3. **Traffic Restoration Phase** (`rolling_deploy_starting: false`)
   - Removes iptables blocking rules
   - Node begins accepting new connections
   - Load balancer resumes forwarding traffic

### Iptables Chains Explained

- **FORWARD chain**: Used for Docker containers with bridge networking
- **INPUT chain**: Used for non-containerized applications or host network mode

## Important Considerations

### Serial Deployment

⚠️ **Critical**: Always use `serial: 1` or small numbers to prevent deploying all nodes simultaneously:

```yaml
- name: Deploy servers
  hosts: app_servers
  serial: "{{ serial_number | default(1) }}"
  # ...
```

This ensures at least some nodes remain available during the deployment.

### Connection Draining

By default, blocking is immediate and unconditional — `DROP` discards packets for already-established connections too, so any in-flight request is cut the instant the block is applied.

If that's a problem (e.g. long-running requests, monitors alerting on the cutover), set `rolling_deploy_graceful_reject_seconds` to give existing connections a chance to finish before the hard block takes effect:

```yaml
rolling_deploy_graceful_reject_seconds: 30
```

This adds up to 30 seconds to the blocking phase itself (it runs before control returns to your playbook), but only new connection attempts are affected during that window — nothing already open gets cut early. After the grace period, the role falls back to `rolling_deploy_iptables_jump` (`DROP` by default) as before.

Note this is different from the external delay pattern below, which pauses *after* the node is already fully blocked (of limited use with the default `DROP` behavior, since anything in-flight has already been terminated by the time this task runs):

```yaml
- name: Wait for connections to drain
  wait_for:
    timeout: 30
```

### Health Checks

Always verify application health before re-enabling traffic:

```yaml
- name: Verify service is ready
  uri:
    url: http://localhost:{{ app_port }}/health
    status_code: 200
  retries: 10
  delay: 5
```

### Multi-Node Clusters

Only use this role when you have multiple nodes (`when: groups['servers'] | length > 1`), otherwise you'll cause a complete outage.

## Troubleshooting

### Check Iptables Rules

```bash
# View IPv4 rules
sudo iptables -L -n -v

# View IPv6 rules
sudo ip6tables -L -n -v
```

### Manual Rule Cleanup

If deployment fails and rules aren't cleaned up:

```bash
# Remove blocking rules
sudo iptables -D INPUT -s <load_balancer_ip> -j DROP
sudo iptables -D FORWARD -s <load_balancer_ip> -j DROP
```

### Verify Role Execution

Run with verbose output:

```bash
ansible-playbook -i inventory playbook.yml -vv
```

## Dependencies

None.

## License

[GNU General Public License v3.0](LICENSE)

## Author Information

Created and maintained by **Ivo Branco** at [FCCN](https://www.fccn.pt/).

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

---

**Repository**: [fccn/ansible-rolling-deploy](https://github.com/fccn/ansible-rolling-deploy)


