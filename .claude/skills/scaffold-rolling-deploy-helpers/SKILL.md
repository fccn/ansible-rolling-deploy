---
name: scaffold-rolling-deploy-helpers
description: Scaffold the reusable `tasks/close_node.yml`, `tasks/open_node.yml`, `tasks/healthcheck.yml`, `group_vars/all/rolling_deploy.yml`, and `rolling_execute.yml` files in an Ansible repository so any service play can adopt zero-downtime rolling deploys with two `import_tasks` lines. Use when starting a new Ansible repo, when an existing repo inlines the rolling-deploy role in every play (DRY violation), or when the user asks to "set up rolling deploy helpers", "add the nau pattern", or "make rolling deploy reusable".
---

# Scaffold rolling-deploy helpers in a repository

This skill creates the shared task files and group vars that let every service playbook in a repository adopt the rolling-deploy pattern with a single `import_tasks: tasks/close_node.yml` / `tasks/open_node.yml` pair. The canonical reference is [fccn/nau_playbooks](https://github.com/fccn/nau_playbooks).

## When this applies

- The repo has (or will have) multiple service playbooks that all need rolling restarts.
- You see `import_role: name: ansible-rolling-deploy` repeated across several plays — that is the smell this skill removes.
- The user is bootstrapping a new ops/playbooks repo and wants the pattern in place before writing service plays.

If the repo only has one play that needs it, prefer the `add-rolling-deploy` skill — do not scaffold helpers for a single caller.

## Prerequisites to check first

1. **Inventory has a load-balancer group.** Find the group name (commonly `balancer_servers`, `load_balancers`, `haproxy`). If none exists, ask the user what to name it and have them populate it before continuing — `group_vars/all/rolling_deploy.yml` references it.
2. **`requirements.yml` includes the role.** Add if missing:
   ```yaml
   - src: git+https://github.com/fccn/ansible-rolling-deploy.git
     version: main  # pin to a commit SHA in production
   ```
3. **Keepalived in use? (optional)** If the repo uses [fccn/ansible-keepalived](https://github.com/fccn/ansible-keepalived) for VIPs, include the priority-override blocks below. If not, omit those tasks — do not pull in keepalived just to satisfy the template.
4. **Healthcheck convention.** Many repos already have a Makefile `healthcheck` target or a `nau_check_urls` role. Match the existing convention rather than inventing a new one.

## Files to create

Substitute placeholders: `<lb_group>` (load-balancer inventory group name), and drop the keepalived blocks if not applicable.

### `group_vars/all/rolling_deploy.yml`

```yaml
---
# Resolve the load balancer IPs once from inventory so every play picks them up automatically.
rolling_deploy_parent_servers_ipv4: "{{ groups['<lb_group>'] | map('extract', hostvars, ['ansible_host']) | list }}"
# Uncomment if the load balancers have IPv6 addresses:
# rolling_deploy_parent_servers_ipv6: "{{ groups['<lb_group>'] | map('extract', hostvars, ['ansible_host_ipv6']) | list }}"
```

### `tasks/close_node.yml`

```yaml
---
# Drain a node before applying changes. Imported by service playbooks.
# Toggle off per-call with: rolling_deploy_enabled: false

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

### `tasks/open_node.yml`

```yaml
---
# Restore a node to rotation after a successful deploy + healthcheck.

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

### `tasks/healthcheck.yml`

Use whichever pattern already exists in the repo. If nothing exists, a reasonable default is:

```yaml
---
- name: Run Makefile healthcheck
  shell: make --jobs 20 --no-print-directory --directory {{ item }} healthcheck
  retries: "{{ healthcheck_retries | default(50) }}"
  delay: "{{ healthcheck_delay | default(30) }}"
  register: result
  until: result.rc == 0
  check_mode: no
  changed_when: false
  with_items: "{{ makefile_healthcheck }}"
  when: makefile_healthcheck is defined
```

Adapt to the project: `uri:` checks against a `/health` endpoint, `wait_for:` on a port, or a custom role like `nau_check_urls`.

### `rolling_execute.yml` (optional but high-value)

A generic playbook that runs any shell command with the close → run → healthcheck → open cycle. Eliminates the need to write a one-off playbook for routine maintenance like `docker pull` or rolling restarts.

```yaml
---
# Run a shell command across a cluster with rolling-restart semantics.
#
# Example:
#   ansible-playbook -i hosts.ini rolling_execute.yml \
#     --limit mongo_docker_servers \
#     -e "command='docker pull mongo:6.0'"

- hosts: all
  serial: "{{ serial_number | default(1) }}"
  become: true
  gather_facts: true
  vars:
    rolling_deploy_enabled: true
  tasks:
    - name: Check that 'command' was provided
      assert:
        that: command is defined
        fail_msg: You need to pass -e command='...'
        quiet: true

    - import_tasks: tasks/close_node.yml

    - name: Run command
      shell: "{{ command }}"
      register: exec_output

    - name: Print stdout
      debug:
        msg: "{{ exec_output.stdout_lines }}"
      when: exec_output.stdout_lines is defined

    - import_tasks: tasks/healthcheck.yml
    - import_tasks: tasks/open_node.yml
```

## How callers use the helpers

Once scaffolded, every service play collapses to:

```yaml
- name: Deploy <service>
  hosts: <service>_servers
  serial: "{{ serial_number | default(1) }}"
  become: true
  gather_facts: true
  tasks:
    - import_tasks: tasks/close_node.yml
      when: (groups['<service>_servers'] | length) > 1

    - import_role:
        name: <service>_deploy

    - import_tasks: tasks/healthcheck.yml

    - import_tasks: tasks/open_node.yml
      when: (groups['<service>_servers'] | length) > 1
```

## Migration from inlined role usage

If the repo already imports `ansible-rolling-deploy` directly in several plays:

1. Create the helper files above.
2. For each play, replace the pre-deploy `import_role: ansible-rolling-deploy` (with `rolling_deploy_starting: true`) with `import_tasks: tasks/close_node.yml`.
3. Replace the post-deploy unblock with `import_tasks: tasks/open_node.yml`.
4. Preserve the existing `when:` guard (typically `(groups['…'] | length) > 1`) on both imports.
5. Move any hardcoded `rolling_deploy_parent_servers_ipv4` lists out of the plays and into `group_vars/all/rolling_deploy.yml`.
6. Run `ansible-playbook ... --check --diff --tags rolling_deploy` to confirm the iptables tasks still fire on the same hosts.

## Common mistakes to avoid

- **Hardcoding the load-balancer group name in `close_node.yml`.** Keep the group reference only in `group_vars/all/rolling_deploy.yml` so the helpers stay reusable.
- **Pulling in keepalived blocks when the repo does not use keepalived.** The `when: keepalived_vrrp_instances is defined` guard makes the tasks safe to keep, but if keepalived is not a dependency at all, drop the blocks entirely and slim `requirements.yml`.
- **Forgetting the `rolling_deploy_enabled` toggle.** Callers (e.g. test environments, single-node deploys) need a way to disable the iptables step without forking the helper.
- **Skipping the `meta: flush_handlers`** in `close_node.yml` — without it, a handler queued earlier in the play can restart the service AFTER traffic is blocked, racing the deploy.
