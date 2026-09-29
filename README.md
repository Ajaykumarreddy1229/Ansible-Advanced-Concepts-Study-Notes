# Ansible-Advanced-Concepts-Study-Notes

## 📚 Topics Covered

* Handlers and Notify
* Blocks
* Rescue
* Always
* Shell, Command and Raw
* Conditions
* LAMP Stack
* Lookups
* Jinja2 Templates
* Ansible Strategies
* Pip Module

---

# 1. Handlers and Notify

## What are Handlers?

Handlers are special Ansible tasks that run only when they are **notified** by another task.

They are useful when a service needs to be:

* Started
* Restarted
* Reloaded

### Basic Flow

```text
Task
  |
  | notify
  v
Handler
  |
  v
Start / Restart Service
```

### Example

```yaml
---
- name: Handlers
  hosts: all

  tasks:
    - name: installing apache
      yum:
        name: httpd
        state: present
      notify: starting apache

  handlers:
    - name: starting apache
      service:
        name: httpd
        state: started
```

Run:

```bash
ansible-playbook handlers.yml
```

### Important Point

A handler runs when the task that notified it reports a change.

---

# 2. Blocks in Ansible

A **block** groups multiple tasks together.

Blocks can be combined with:

* `block`
* `rescue`
* `always`

This provides error-handling functionality.

### Programming Comparison

| Programming | Ansible |
| ----------- | ------- |
| try         | block   |
| catch       | rescue  |
| finally     | always  |

### block

Contains the main tasks.

### rescue

Runs when a task inside the block fails.

### always

Runs whether the block succeeds or fails.

---

# 3. Block + Rescue + Always

Example structure:

```yaml
tasks:

  - block:

      - name: Install Nginx
        dnf:
          name: nginx
          state: present

      - name: Deploy index.html
        copy:
          content: "{{ webpage }}"
          dest: /usr/share/nginx/html/index.html

    rescue:

      - name: Display Failure Message
        debug:
          msg: "Deployment failed. Starting rollback..."

    always:

      - name: Deployment Status
        debug:
          msg: "Deployment process completed."
```

### Typical Uses

**Block**

* Group related deployment tasks

**Rescue**

* Rollback changes
* Remove partially installed software
* Restore backups
* Send alerts

**Always**

* Collect logs
* Display deployment status
* Send notifications
* Cleanup temporary files

---

# 4. Handlers + Notify + Block

These concepts can be combined to create more reliable deployments.

Example workflow:

```text
Ansible Playbook
       |
       v
     Block
       |
       +------> Install Application
       |
       +------> Deploy Configuration
       |
       +------> Start Service
       |
       +------> notify
       |
       v
    Handler
       |
       v
 Restart Service

If something fails
       |
       v
    Rescue
       |
       v
   Rollback

Regardless of result
       |
       v
     Always
       |
       v
 Status / Logs / Cleanup
```

---

# 5. Shell, Command and Raw

Ansible can execute Linux commands using different modules.

## Shell

Used to execute shell commands.

```yaml
- name: installing apache
  shell: yum install httpd -y
```

## Command

Used to execute commands without shell-specific features.

```yaml
- name: installing git
  command: yum install git -y
```

## Raw

Can execute commands directly on remote systems.

```yaml
- name: installing maven
  raw: yum install maven -y
```

### Quick Comparison

| Module    | Usage                                           |
| --------- | ----------------------------------------------- |
| `command` | Use whenever possible                           |
| `shell`   | Use when shell features are required            |
| `raw`     | Useful for bootstrapping systems without Python |

---

# 6. Conditions

Conditions allow Ansible to execute tasks depending on specific conditions.

The `when` keyword is used for this.

### RedHat Example

```yaml
- hosts: all

  tasks:

    - name: installing apache on RedHat
      yum:
        name: httpd
        state: present
      when: ansible_os_family == "RedHat"
```

### Ubuntu Example

```yaml
- hosts: all

  tasks:

    - name: installing apache on Ubuntu
      apt:
        name: apache2
        state: present
      when: ansible_os_family == "Debian"
```

### Check Facts

```bash
ansible all -m setup
```

Search for OS family:

```bash
ansible all -m setup | grep -i family
```

---

# 7. Conditional Installation on a Specific Node

A task can also be executed only on a particular node.

Example:

```yaml
- hosts: all
  gather_facts: false

  tasks:

    - name: installing apache
      yum:
        name: httpd
        state: present
      when: ansible_nodename == "dev-1"
```

---

# 8. Cluster Concepts

### Cluster

A group of servers/nodes that communicate with each other.

### Homogeneous

Servers with the same OS and flavor.

### Heterogeneous

Servers with different OS or flavors.

### Package Managers

```text
RedHat  -> yum
Ubuntu  -> apt
Python  -> pip
```

---

# 9. LAMP Stack

LAMP stands for:

```text
L → Linux
A → Apache
M → MySQL
P → PHP / Python
```

Example:

```yaml
---
- name: LAMP
  hosts: all

  tasks:

    - name: Installing Apache
      yum:
        name: httpd
        state: present

    - name: Installing MySQL
      yum:
        name: mysql
        state: present

    - name: Installing Python
      yum:
        name: python3
        state: present
```

---

# 10. Lookups

The Ansible `lookup` functionality can be used to read data from sources such as files.

Example:

```yaml
---
- name: Lookups
  hosts: all

  vars:
    creds: "{{ lookup('file', '/root/creds.txt') }}"

  tasks:

    - debug:
        msg: "My Credentials are {{ creds }}"
```

### Important

Avoid storing real passwords or secrets directly in playbooks.

For production environments, use appropriate secret-management mechanisms.

---

# 11. Jinja2 Templates

Jinja2 is a templating engine used with Ansible to dynamically generate:

* Configuration files
* Scripts
* Application files

Variables are represented using:

```text
{{ variable_name }}
```

Example:

```text
listen {{ nginx_port }};
server_name {{ server_name }};
root {{ web_root }};
```

---

# 12. Jinja2 Nginx Example

Create a templates directory:

```bash
mkdir templates
```

Create:

```text
templates/nginx.conf.j2
```

Example:

```nginx
server {
    listen {{ nginx_port }};
    server_name {{ server_name }};

    location / {
        root {{ web_root }};
        index index.html;
    }
}
```

Playbook:

```yaml
---
- name: Deploy Nginx Config using Jinja2
  hosts: all
  become: yes

  vars:
    nginx_port: 80
    server_name: mywebsite.com
    web_root: /usr/share/nginx/html/

  tasks:

    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Copy Nginx Config with Template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx

  handlers:

    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

Run:

```bash
ansible-playbook nginx.yml
```

---

# 13. Jinja2 Apache Example

Template:

```text
templates/httpd.conf.j2
```

Example:

```apache
Listen {{ http_port }}

<VirtualHost *:{{ http_port }}>
    ServerName {{ server_name }}
    DocumentRoot {{ document_root }}

    <Directory "{{ document_root }}">
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/{{ server_name }}_error.log
    CustomLog /var/log/httpd/{{ server_name }}_access.log combined
</VirtualHost>
```

Variables:

```yaml
vars:
  http_port: 80
  server_name: example.com
  document_root: /var/www/html
```

The template module copies the Jinja2 template from the control node to the managed node and replaces the variables with their values.

---

# 14. Ansible Strategies

A strategy determines how Ansible executes tasks across multiple managed nodes.

## Linear

Linear is the default strategy.

```text
Task 1
 ├── Node 1
 ├── Node 2
 └── Node 3
       |
       v
Task 2
 ├── Node 1
 ├── Node 2
 └── Node 3
```

Ansible waits for all hosts to complete the current task before moving to the next task.

Example:

```yaml
- name: Strategies
  hosts: all
  strategy: Linear

  tasks:

    - name: Installing Apache
      yum:
        name: httpd
        state: present
```

## Free

The `free` strategy allows hosts to progress independently instead of waiting for all hosts to finish each task.

```text
Node 1     Node 2     Node 3

Task 1     Task 1     Task 1
  ↓          ↓          ↓
Task 2     Task 2     Task 2
  ↓          ↓          ↓
Task 3     Task 3     Task 3
```

Useful when servers can be handled independently and faster execution is desired.

## Other Strategies

```text
host_pinned
debug
custom strategy
```

---

# 15. Pip Module

`pip` is a package manager used for installing Python libraries/modules.

Example:

```yaml
- name: Playbook using pip
  hosts: all

  tasks:

    - name: install pip module
      yum:
        name: pip
        state: present

    - name: installing NumPy
      pip:
        name: NumPy
        state: present

    - name: installing Pandas
      pip:
        name: Pandas
        state: present
```

---

# 🔥 Key Takeaways

```text
Handlers
   ↓
Run only when notified

Notify
   ↓
Triggers handlers

Block
   ↓
Groups related tasks

Rescue
   ↓
Handles failures

Always
   ↓
Runs regardless of success/failure

Conditions
   ↓
Control task execution

Jinja2
   ↓
Create dynamic configuration files

Strategies
   ↓
Control task execution across hosts

Lookups
   ↓
Read external data

Pip
   ↓
Install Python packages
```

## 🚀 Today's Learning

Today I learned how Ansible can be used beyond basic task automation to build more reliable and dynamic deployments using **Handlers, Notify, Blocks, Rescue, Always, Conditions, Jinja2 Templates, Strategies, Lookups, Shell/Command/Raw modules, LAMP automation, and Pip**.
