# Ansible Role: ansible-rolling-deploy

An Ansible role to apply a rolling deploy, a blue/green deployment, for a cluster of nodes.
This role can mark a node for maintenance and also readd it to the cluster.
The role uses iptables to block upstream load balancer connections.

## Requirements

- Ansible 2.9 or higher
- a Linux distribution that uses iptables

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

Or define explicitly:
```yaml
rolling_deploy_parent_servers_ipv4:
- 172.24.1.81
- myloadbalancer.priv.fccn.pt
```

Or use the inventory itself if you manage the load balancers:

```ini
[load_balancers_servers]
lb01-dev.myservice.fccn.pt   ansible_host=172.24.1.81
```

```yaml
rolling_deploy_parent_servers_ipv4: "{{ groups['load_balancers_servers'] | map('extract', hostvars, ['ansible_host']) | list }}"
```


## Dependencies

None.

## Example Playbook

### Block Load Balancer Connections at Start

To block a node from receiving new load balancer connections using iptables:

```yaml
- name: Start rolling deploy - block load balancer connections
  import_role:
    name: rolling_deploy
  vars:
    rolling_deploy_starting: true
```

### Unblock Load Balancer Connections

To re-enable load balancer connections after deployment:

```yaml
- name: Finish rolling deploy - unblock load balancer connections
  import_role:
    name: rolling_deploy
  vars:
    rolling_deploy_starting: false
```


## Example playbook

To perform a rolling deployment:

```yaml


- name: Deploy financial manager servers
  hosts: financial_manager_docker_servers
  serial: "{{ serial_number | default(1) }}" # deploy in sequence
  become: True
  gather_facts: True
  tasks:
    - name: Start rolling deploy - block load balancer connections
      import_role:
        name: rolling_deploy
      vars:
        rolling_deploy_docker: true
        rolling_deploy_starting: true
      when: ( groups['financial_manager_docker_servers'] | length ) > 1 and ( financial_manager_docker_servers | default(false) | bool )

    - name: Install or upgrade docker daemon
      import_role:
        name: geerlingguy.docker
      when: financial_manager_deploy | default(false) | bool

    - import_role:
        name: snmpd
      tags: snmpd
      when: financial_manager_deploy | default(false) | bool
    - import_role:
        name: snmpd_docker
      tags: snmpd
      when: financial_manager_deploy | default(false) | bool

    - name: Deploy financial manager app
      import_role:
        name: financial_manager_docker_deploy
      when: financial_manager_deploy | default(false) | bool
      vars:
        # MySQL DB
        financial_manager_mysql_docker_hostname: "{{ hostvars[groups['financial_manager_mysql_docker_servers'][0]].ansible_host }}"
        financial_manager_mysql_docker_port: "{{ hostvars[groups['financial_manager_mysql_docker_servers'][0]].financial_manager_mysql_docker_port }}"
        financial_manager_mysql_database: "{{ hostvars[groups['financial_manager_mysql_docker_servers'][0]].financial_manager_mysql_database }}"
        financial_manager_mysql_user: "{{ hostvars[groups['financial_manager_mysql_docker_servers'][0]].financial_manager_mysql_user }}"
        financial_manager_mysql_password: "{{ hostvars[groups['financial_manager_mysql_docker_servers'][0]].financial_manager_mysql_password }}"
        financial_manager_mysql_root_password: "{{ hostvars[groups['financial_manager_mysql_docker_servers'][0]].financial_manager_mysql_root_password }}"

        # Django Cache
        financial_manager_caches_default_redis_host: "{{ hostvars[groups['redis_docker_servers'][0]].redis_virtual_ipv4 }}"
        financial_manager_caches_default_redis_port: "{{ hostvars[groups['redis_docker_servers'][0]].redis_docker_port }}"
        financial_manager_caches_default_redis_db:   "8"

        # Celery broker
        financial_manager_celery_broker_redis_host: "{{ hostvars[groups['redis_docker_servers'][0]].ansible_host }}"
        financial_manager_celery_broker_redis_port: "{{ hostvars[groups['redis_docker_servers'][0]].redis_docker_port }}"
        financial_manager_celery_broker_redis_db:   "9"

    - name: End rolling deploy - open load balancer connections
      import_role:
        name: rolling_deploy
      vars:
        rolling_deploy_docker: true
        rolling_deploy_starting: false
      when: ( groups['financial_manager_docker_servers'] | length ) > 1 and ( financial_manager_docker_servers | default(false) | bool )

```


```bash
ansible-playbook -i inventory playbook.yml
```

## License

GPLv3

## Author Information

Created by Ivo Branco
