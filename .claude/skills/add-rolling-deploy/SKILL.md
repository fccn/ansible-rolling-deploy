---
name: add-rolling-deploy
description: Add the fccn/ansible-rolling-deploy role to an existing Ansible playbook so a multi-node service can be deployed without downtime. Use when a play targets a cluster of >1 nodes behind load balancers and currently restarts all nodes at once, or when the user asks to "add rolling deploy", "make this zero-downtime", "drain traffic before deploy", or "block load balancer during update".
---

# Add rolling deploy to an Ansible playbook

This skill wires the `fccn.ansible_rolling_deploy` role into an existing play so that each node is drained at the load balancer (via iptables DROP rules) before the deploy tasks run, and reopened afterwards.

## When this applies

Apply this skill when ALL of the following are true:
- The play targets a host group with more than one node (a cluster).
- Traffic reaches those nodes via load balancers (HAProxy, nginx, keepalived VIP, etc.) whose IPs are reachable from the nodes.
- The deploy step inside the play causes a restart, image pull, or other window where the node should not receive new traffic.

Do NOT apply if there is only one node — blocking traffic on a single-node service is a full outage. Guard with `when: (groups['<group>'] | length) > 1`.

## Prerequisites to check first

1. **Role available**: confirm `requirements.yml` already references the role. If not, add:
   ```yaml
   - src: git+https://github.com/fccn/ansible-rolling-deploy.git
     version: main  # pin to a commit SHA in production
   ```
   and remind the user to run `ansible-galaxy install -r requirements.yml -p vendor/roles` (or wherever their `roles_path` points).

2. **Load balancer IPs known**: the role needs `rolling_deploy_parent_servers_ipv4` (and optionally `_ipv6`). Prefer resolving from inventory rather than hardcoding. The canonical pattern is in `group_vars/all/rolling_deploy.yml`:
   ```yaml
   rolling_deploy_parent_servers_ipv4: "{{ groups['balancer_servers'] | map('extract', hostvars, ['ansible_host']) | list }}"
   ```
   If that file does not exist, create it (substituting the actual load-balancer inventory group name).

3. **Serial deployment**: the play MUST use `serial:` to update one node (or a small batch) at a time. Add `serial: "{{ serial_number | default(1) }}"` if missing.

## How to wire it in

Two patterns. Pick based on whether the repo already has `tasks/close_node.yml` / `tasks/open_node.yml` helpers (the nau_playbooks convention).

### Pattern A — repo has close/open helpers

Just import them around the deploy task:

```yaml
- name: Deploy <service>
  hosts: <service>_servers
  serial: "{{ serial_number | default(1) }}"
  become: true
  gather_facts: true
  tasks:
    - import_tasks: tasks/close_node.yml
      when: (groups['<service>_servers'] | length) > 1

    - name: Deploy <service>
      import_role:
        name: <service>_deploy
      when: <service>_deploy | default(false) | bool

    - import_tasks: tasks/healthcheck.yml

    - import_tasks: tasks/open_node.yml
      when: (groups['<service>_servers'] | length) > 1
```

If the helpers do not exist, scaffold them with the `scaffold-rolling-deploy-helpers` skill first — do not inline the role twice per play.

### Pattern B — inline role import (smaller repos)

```yaml
- name: Deploy <service>
  hosts: <service>_servers
  serial: "{{ serial_number | default(1) }}"
  become: true
  gather_facts: true
  tasks:
    - name: Start rolling deploy — block load balancer connections
      import_role:
        name: ansible-rolling-deploy
      vars:
        rolling_deploy_starting: true
      when: (groups['<service>_servers'] | length) > 1 and (<service>_deploy | default(false) | bool)

    - name: Deploy <service>
      import_role:
        name: <service>_deploy
      when: <service>_deploy | default(false) | bool

    - name: Verify application health
      uri:
        url: "http://localhost:{{ <service>_port }}/health"
        status_code: 200
      retries: 10
      delay: 5

    - name: End rolling deploy — open load balancer connections
      import_role:
        name: ansible-rolling-deploy
      vars:
        rolling_deploy_starting: false
      when: (groups['<service>_servers'] | length) > 1 and (<service>_deploy | default(false) | bool)
```

## Common mistakes to avoid

- **Missing the closing import.** If the play fails between block and unblock, the iptables DROP rule survives and the node stays out of rotation. Either run a follow-up `ansible-playbook ... -e rolling_deploy_starting=false`, or document the manual `iptables -D` recovery.
- **Forgetting the `length > 1` guard.** On single-node test envs, the role will black-hole the only node.
- **No health check between block and unblock.** Add either a `uri:` check, a `wait_for:` on the service port, or `import_tasks: tasks/healthcheck.yml` — otherwise you reopen traffic to a broken node.
- **Hardcoding load balancer IPs in the play.** Resolve from inventory via `group_vars/all/rolling_deploy.yml` so the same playbook works across environments.
- **Replacing instead of importing.** Use `import_role` (static) rather than `include_role` so tags propagate predictably.

## What to verify when done

1. The play has `serial:` set.
2. Block and unblock both have the same `when:` guard so they never get out of step.
3. Run `ansible-playbook ... --check --diff` on a staging inventory and confirm the iptables tasks appear for the right hosts.
4. After a real run, on one of the deployed nodes: `sudo iptables -L INPUT -n | grep DROP` should show no leftover rules from the deploy.
