# ⚙️ Ansible — DevOps Interview Guide

> [← Terraform](./Terraform.md) | [Main Index](./README.md) | [AWS →](./AWS.md)

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Inventory](#inventory)
3. [Playbooks](#playbooks)
4. [Roles](#roles)
5. [Variables & Templates](#variables--templates)
6. [Ansible Vault](#ansible-vault)
7. [Dynamic Inventory](#dynamic-inventory)
8. [Error Handling](#error-handling)
9. [Interview Questions](#interview-questions)
10. [Scenario-Based Questions](#scenario-based-questions)
11. [Hands-On Labs](#hands-on-labs)
12. [Cheat Sheet](#cheat-sheet)

---

## Core Concepts

### How Ansible Works

```
Control Node                  Managed Nodes
(your laptop/CI)              (servers to configure)

  ┌─────────────┐
  │  Inventory  │──── SSH / WinRM ────> Node 1 (web01)
  │  Playbooks  │──── SSH / WinRM ────> Node 2 (web02)
  │  Roles      │──── SSH / WinRM ────> Node 3 (db01)
  └─────────────┘
        │
   Python + SSH
   No agent needed!
   Idempotent tasks
```

### Key Principles

| Principle | Description |
|-----------|-------------|
| **Agentless** | Uses SSH/WinRM, no agent installed on nodes |
| **Idempotent** | Running a playbook multiple times = same result |
| **Declarative** | Describe desired state; Ansible figures out how |
| **Push-based** | Control node pushes to managed nodes |
| **YAML-based** | Human-readable configuration |

---

## Inventory

### Static Inventory

```ini
# inventory/hosts.ini
[webservers]
web01.example.com
web02.example.com ansible_host=10.0.1.12

[dbservers]
db01.example.com ansible_host=10.0.2.10 ansible_user=ubuntu

[loadbalancers]
lb01.example.com

# Group of groups
[production:children]
webservers
dbservers
loadbalancers

[production:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_key
ansible_python_interpreter=/usr/bin/python3

[webservers:vars]
http_port=80
max_connections=200
```

### YAML Inventory

```yaml
# inventory/hosts.yml
all:
  vars:
    ansible_user: deploy
    ansible_ssh_private_key_file: ~/.ssh/deploy_key

  children:
    webservers:
      hosts:
        web01:
          ansible_host: 10.0.1.11
        web02:
          ansible_host: 10.0.1.12
      vars:
        http_port: 80

    dbservers:
      hosts:
        db01:
          ansible_host: 10.0.2.10
          db_port: 5432
        db02:
          ansible_host: 10.0.2.11
          db_port: 5432

    staging:
      children:
        webservers:
        dbservers:
```

---

## Playbooks

### Production-Grade Playbook

```yaml
---
# site.yml — Main playbook
- name: Configure and deploy web application
  hosts: webservers
  become: yes
  gather_facts: yes
  serial: "25%"          # Rolling update: 25% of hosts at a time

  pre_tasks:
    - name: Notify monitoring — start maintenance window
      uri:
        url: "https://monitoring.example.com/api/maintenance"
        method: POST
        body_format: json
        body:
          host: "{{ inventory_hostname }}"
          duration: 600
        headers:
          Authorization: "Bearer {{ monitoring_api_token }}"
      delegate_to: localhost
      run_once: false

    - name: Wait for server to be reachable
      wait_for_connection:
        timeout: 60

  roles:
    - role: common
    - role: nginx
      vars:
        nginx_worker_processes: "{{ ansible_processor_vcpus }}"
    - role: app_deploy
      tags: [deploy]

  post_tasks:
    - name: Verify application health
      uri:
        url: "http://{{ ansible_host }}/health"
        status_code: 200
      register: health_check
      retries: 5
      delay: 10
      until: health_check.status == 200

    - name: Remove from maintenance window
      uri:
        url: "https://monitoring.example.com/api/maintenance/{{ inventory_hostname }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ monitoring_api_token }}"
      delegate_to: localhost

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted

    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

### Common Modules

```yaml
# Package management
- name: Install packages
  package:
    name:
      - nginx
      - python3-pip
      - git
    state: present
    update_cache: yes

# Service management
- name: Start and enable nginx
  service:
    name: nginx
    state: started
    enabled: yes

# File operations
- name: Create config directory
  file:
    path: /etc/myapp
    state: directory
    owner: www-data
    group: www-data
    mode: '0750'

- name: Copy config file
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: Reload nginx

- name: Render template
  template:
    src: templates/app.conf.j2
    dest: /etc/myapp/app.conf
    owner: www-data
    mode: '0640'
  notify: Restart app

# User management
- name: Create deploy user
  user:
    name: deploy
    shell: /bin/bash
    groups: docker
    append: yes
    create_home: yes

# SSH authorized key
- name: Add deploy SSH key
  authorized_key:
    user: deploy
    state: present
    key: "{{ deploy_public_key }}"

# Execute commands
- name: Run database migrations
  command: /app/bin/migrate
  args:
    chdir: /app
  environment:
    DATABASE_URL: "{{ db_url }}"
  register: migration_output
  changed_when: "'Nothing to migrate' not in migration_output.stdout"

# Git clone/pull
- name: Clone application repository
  git:
    repo: "https://github.com/org/myapp.git"
    dest: /app
    version: "{{ app_version }}"
    force: yes

# Systemd unit
- name: Create systemd service
  template:
    src: templates/myapp.service.j2
    dest: /etc/systemd/system/myapp.service
  notify:
    - Reload systemd
    - Restart myapp

# Wait for port
- name: Wait for app to start
  wait_for:
    port: 8080
    host: "{{ ansible_host }}"
    timeout: 60

# Lineinfile
- name: Update sshd config
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
    validate: /usr/sbin/sshd -t -f %s
  notify: Restart sshd

# Blockinfile
- name: Add Nginx virtual host
  blockinfile:
    path: /etc/nginx/sites-available/myapp
    block: |
      server {
          listen 80;
          server_name {{ domain_name }};
          root /var/www/{{ app_name }};
      }
    create: yes
    marker: "# {mark} ANSIBLE MANAGED BLOCK"
```

---

## Roles

### Role Structure

```
roles/
└── nginx/
    ├── tasks/
    │   ├── main.yml          # Entry point for tasks
    │   ├── install.yml       # Installation tasks
    │   └── configure.yml     # Configuration tasks
    ├── handlers/
    │   └── main.yml          # Handlers (triggered by notify)
    ├── templates/
    │   └── nginx.conf.j2     # Jinja2 templates
    ├── files/
    │   └── htpasswd          # Static files
    ├── vars/
    │   └── main.yml          # Role variables (high priority)
    ├── defaults/
    │   └── main.yml          # Default variables (lowest priority)
    ├── meta/
    │   └── main.yml          # Role metadata + dependencies
    └── README.md
```

### roles/nginx/tasks/main.yml

```yaml
---
- name: Install nginx
  package:
    name: nginx
    state: present
  tags: [install]

- name: Create nginx directories
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled
    - /var/log/nginx
  tags: [configure]

- name: Deploy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
    validate: nginx -t -c %s
  notify: Reload nginx
  tags: [configure]

- name: Enable and start nginx
  service:
    name: nginx
    state: started
    enabled: yes
  tags: [service]
```

### roles/nginx/defaults/main.yml

```yaml
---
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_client_max_body_size: 10m
nginx_log_format: combined
nginx_access_log: /var/log/nginx/access.log
nginx_error_log: /var/log/nginx/error.log
nginx_gzip: yes
nginx_server_names_hash_bucket_size: 64
```

### roles/nginx/templates/nginx.conf.j2

```nginx
user www-data;
worker_processes {{ nginx_worker_processes }};
error_log {{ nginx_error_log }} warn;
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log {{ nginx_access_log }} main;
    keepalive_timeout {{ nginx_keepalive_timeout }};

{% if nginx_gzip %}
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript;
{% endif %}

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## Variables & Templates

### Variable Precedence (highest → lowest)

```
1.  Extra vars: -e key=value
2.  task vars (in task)
3.  block vars
4.  role and include vars
5.  play vars_files
6.  play vars_prompt
7.  play vars
8.  set_facts / registered vars
9.  host facts
10. playbook host_vars/*
11. playbook group_vars/*
12. inventory host_vars/*
13. inventory group_vars/*
14. inventory group_vars/all
15. playbook group_vars/all
16. role defaults
17. command line values (lowest)
```

### Jinja2 Template Features

```jinja2
{# Comment #}
{{ variable }}                        {# Variable output #}
{{ variable | default('fallback') }}  {# Default filter #}
{{ variable | upper }}                {# Uppercase filter #}
{{ list | join(', ') }}               {# Join list #}
{{ dict | to_json }}                  {# Serialize to JSON #}
{{ "hello" | hash('sha1') }}          {# Hash #}

{# Conditionals #}
{% if ansible_os_family == 'Debian' %}
apt-get install -y nginx
{% elif ansible_os_family == 'RedHat' %}
yum install -y nginx
{% endif %}

{# Loops #}
{% for server in groups['webservers'] %}
server {{ hostvars[server]['ansible_host'] }};
{% endfor %}

{# Set variables #}
{% set workers = ansible_processor_vcpus * 2 %}
worker_processes {{ workers }};
```

---

## Ansible Vault

```bash
# Encrypt a file
ansible-vault encrypt secrets.yml
ansible-vault encrypt_string 'mysecret' --name 'db_password'

# Edit encrypted file
ansible-vault edit secrets.yml

# View encrypted file
ansible-vault view secrets.yml

# Decrypt file
ansible-vault decrypt secrets.yml

# Rekey (change password)
ansible-vault rekey secrets.yml

# Run playbook with vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass
ansible-playbook site.yml --vault-id prod@.vault_pass_prod

# Multiple vault IDs
ansible-playbook site.yml \
  --vault-id staging@staging-pass.txt \
  --vault-id production@production-pass.txt

# Encrypted vars in group_vars/all/vault.yml
$ANSIBLE_VAULT;1.1;AES256
66386439653236336462626566653065326...
```

---

## Dynamic Inventory

### AWS EC2 Dynamic Inventory

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - eu-west-1
filters:
  instance-state-name: running
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: tags.Environment
    prefix: env
  - key: placement.availability_zone
hostnames:
  - private-ip-address
compose:
  ansible_host: private_ip_address
```

```bash
# Test dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# Use in playbook
ansible-playbook -i inventory/aws_ec2.yml site.yml
```

---

## Error Handling

```yaml
# ignore_errors — continue despite failure
- name: Stop old service (may not exist)
  service:
    name: old-service
    state: stopped
  ignore_errors: yes

# failed_when — custom failure condition
- name: Run health check
  command: /opt/app/health-check.sh
  register: health
  failed_when:
    - health.rc != 0
    - "'WARNING' not in health.stdout"

# changed_when — control when changed is reported
- name: Check git status
  command: git status --porcelain
  register: git_status
  changed_when: git_status.stdout != ""

# block / rescue / always (try/catch/finally)
- block:
    - name: Attempt risky operation
      command: /opt/migrate.sh
      register: migration

    - name: Verify migration
      command: /opt/verify.sh

  rescue:
    - name: Rollback migration
      command: /opt/rollback.sh

    - name: Alert team
      mail:
        to: "ops@example.com"
        subject: "Migration FAILED on {{ inventory_hostname }}"
        body: "{{ migration.stderr }}"

  always:
    - name: Remove migration lock file
      file:
        path: /tmp/migration.lock
        state: absent
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between Ansible and Terraform?**

> **Ansible** is a **configuration management** and **application deployment** tool — it configures what's *inside* servers (packages, files, services, users). **Terraform** is **infrastructure provisioning** — it creates the servers, networks, and cloud resources themselves. They're complementary: Terraform provisions the EC2 instance; Ansible configures nginx and deploys the app onto it. Terraform is declarative with state; Ansible is procedural (tasks run in order) but idempotent.

**Q2: What is idempotency in Ansible?**

> An operation is idempotent if running it multiple times produces the same result as running it once. Ansible modules are designed to be idempotent: `package: state=present` checks if the package is installed; if yes, does nothing; if no, installs it. This means you can safely re-run playbooks for drift correction or new servers without causing unintended side effects.

**Q3: What is the difference between `vars` and `defaults` in a role?**

> `defaults/main.yml` has the **lowest priority** — easily overridden by inventory vars, playbook vars, or extra vars. Use for sensible defaults users might want to change. `vars/main.yml` has **higher priority** — harder to override, not intended for user customization. Use for role-internal constants.

---

### 🟡 Intermediate

**Q4: How do handlers work in Ansible?**

> Handlers are tasks that run only when **notified** by other tasks, and only **once** at the end of the play even if notified multiple times. Use case: restart nginx only after any config change, not after every individual config task. Handlers run in the order they are defined, not in the order they are notified. `meta: flush_handlers` forces handlers to run immediately.

**Q5: Explain Ansible facts and how to use them.**

> Facts are variables automatically gathered about managed nodes by the `setup` module (runs implicitly at play start via `gather_facts: yes`). They include: OS family, distribution, hostname, IPs, CPU count, memory, disk info, etc. Access via `ansible_*` variables: `{{ ansible_os_family }}`, `{{ ansible_processor_vcpus }}`. Use `set_fact` to create custom facts. Cache facts with `fact_caching = jsonfile` in `ansible.cfg` for performance.

**Q6: What is `delegate_to` and when would you use it?**

> `delegate_to` runs a task on a different host than the current play target. Common uses:
> - Remove a host from a load balancer before updating it (run on LB or localhost)
> - Register a DNS record via API from localhost
> - Add/remove from monitoring from a monitoring host
> - `run_once: true` + `delegate_to: localhost` = run exactly once on control node

---

### 🔴 Advanced

**Q7: How would you test Ansible roles?**

> - **Molecule**: Framework for testing Ansible roles. Creates instances (Docker, Vagrant, cloud), runs the role, runs `verifier` tests (TestInfra, Goss, Ansible assert), destroys instances.
> ```bash
> molecule init role myrole
> molecule test    # full cycle: create, converge, verify, destroy
> molecule converge  # just apply role
> molecule verify    # just run tests
> ```
> - **Ansible-lint**: Static analysis of playbooks/roles for best practices
> - **yamllint**: YAML syntax validation
> - CI: Run molecule in Docker executor in Jenkins/GitHub Actions

**Q8: How do you optimize Ansible playbook performance?**

> - `gather_facts: no` when facts not needed (saves ~1-2s per host)
> - `fact_caching = jsonfile` — cache facts between runs
> - Increase `forks` in `ansible.cfg` (default 5, try 20-50)
> - `pipelining = True` — reduces SSH connections per task
> - Use `async` + `poll` for long-running tasks (parallel execution)
> - `strategy: free` — hosts don't wait for each other between tasks
> - Avoid `command`/`shell` modules when idempotent modules exist
> - Use `delegate_to: localhost` for API calls instead of per-host

---

## Scenario-Based Questions

### 🔵 Scenario 1: Rolling Update with Zero Downtime

```yaml
# playbook: rolling_deploy.yml
- hosts: webservers
  serial: 1              # One server at a time
  max_fail_percentage: 0 # Abort if any server fails

  pre_tasks:
    - name: Disable in HAProxy
      haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        socket: /run/haproxy/admin.sock
      delegate_to: "{{ groups['loadbalancers'][0] }}"

  roles:
    - role: app_deploy

  post_tasks:
    - name: Verify app is healthy
      uri:
        url: "http://{{ ansible_host }}:8080/health"
        status_code: 200
      retries: 6
      delay: 10

    - name: Re-enable in HAProxy
      haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        socket: /run/haproxy/admin.sock
      delegate_to: "{{ groups['loadbalancers'][0] }}"
```

---

## Hands-On Labs

### Lab 1: Install and Configure NGINX

```bash
# Install Ansible
pip install ansible

# Create project structure
mkdir ansible-lab && cd ansible-lab
mkdir -p {roles/nginx/{tasks,handlers,templates,defaults},inventory}

# inventory/hosts.ini
cat > inventory/hosts.ini << 'EOF'
[webservers]
localhost ansible_connection=local
EOF

# roles/nginx/tasks/main.yml
cat > roles/nginx/tasks/main.yml << 'EOF'
---
- name: Install nginx
  package:
    name: nginx
    state: present

- name: Start nginx
  service:
    name: nginx
    state: started
    enabled: yes
EOF

# site.yml
cat > site.yml << 'EOF'
---
- hosts: webservers
  become: yes
  roles:
    - nginx
EOF

# Run
ansible-playbook -i inventory/hosts.ini site.yml
ansible-playbook -i inventory/hosts.ini site.yml  # Run again — should show 0 changes
```

---

## Cheat Sheet

```bash
# ─── AD-HOC COMMANDS ──────────────────────────────────
ansible all -m ping
ansible webservers -m setup                    # Gather facts
ansible webservers -m service -a "name=nginx state=restarted" --become
ansible all -m shell -a "df -h"
ansible all -m copy -a "src=file dest=/tmp/file"

# ─── PLAYBOOK ─────────────────────────────────────────
ansible-playbook site.yml
ansible-playbook site.yml -i inventory/prod
ansible-playbook site.yml --tags deploy
ansible-playbook site.yml --skip-tags test
ansible-playbook site.yml --limit webservers
ansible-playbook site.yml --check            # Dry run
ansible-playbook site.yml --diff             # Show file diffs
ansible-playbook site.yml -e "version=2.0"  # Extra vars
ansible-playbook site.yml -v/-vv/-vvv        # Verbosity

# ─── INVENTORY ────────────────────────────────────────
ansible-inventory --list
ansible-inventory --graph
ansible-inventory -i aws_ec2.yml --list

# ─── VAULT ────────────────────────────────────────────
ansible-vault encrypt vars/secrets.yml
ansible-vault edit vars/secrets.yml
ansible-playbook site.yml --vault-password-file .vault_pass

# ─── GALAXY ───────────────────────────────────────────
ansible-galaxy role init myrole
ansible-galaxy install geerlingguy.nginx
ansible-galaxy collection install community.docker
ansible-galaxy install -r requirements.yml
```

---

> **Cross-links:** [← Terraform](./Terraform.md) | [AWS →](./AWS.md) | [CI/CD (Ansible in pipelines) →](./CICD.md) | [Security (hardening with Ansible) →](./Security.md)
