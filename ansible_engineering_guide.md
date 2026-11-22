# Comprehensive Ansible Engineering Guide (`learn-ansible`)

An expanded, deep-dive reference manual covering your entire lab progression (`01-playbook.yml` through `22-shellvCommand.yml`), architectural mechanics, execution lifecycles, and troubleshooting notes.

---

## Table of Contents
1. [Ansible Architecture & Execution Lifecycle](#1-ansible-architecture--execution-lifecycle)
2. [Core CLI Commands & Pre-Flight Validation](#2-core-cli-commands--pre-flight-validation)
3. [Configuration Precedence (`ansible.cfg`)](#3-configuration-precedence-ansiblecfg)
4. [Inventory Design (`inventory.ini`)](#4-inventory-design-inventoryini)
5. [Playbook Structure: Single vs. Multi-Play (`01` to `04`)](#5-playbook-structure-single-vs-multi-play-01-to-04)
6. [Privilege Escalation (`become`) & SSH Authentication](#6-privilege-escalation-become--ssh-authentication)
7. [Data Types & Jinja2 Templating (`13-datatypes.yml`)](#7-data-types--jinja2-templating-13-datatypesyml)
8. [Variables: Sources & Precedence Hierarchy (`05` to `11`)](#8-variables-sources--precedence-hierarchy-05-to-11)
9. [Facts Gathering (`12-gather_facts.yml`)](#9-facts-gathering-12-gather_factsyml)
10. [Conditionals (`when`) & Loops (`loop`)](#10-conditionals-when--loops-loop)
11. [Jinja2 Filters (`17-filters.yml`, `18-filters.yml`)](#11-jinja2-filters-17-filtersyml-18-filtersyml)
12. [Error Handling, `register`, & Idempotency (`19` to `21`)](#12-error-handling-register--idempotency-19-to-21)
13. [`command` vs. `shell` Module (`22-shellvCommand.yml`)](#13-command-vs-shell-module-22-shellvcommandyml)
14. [Lab Troubleshooting Reference Matrix](#14-lab-troubleshooting-reference-matrix)

---

## 1. Ansible Architecture & Execution Lifecycle

When you run `ansible-playbook`, Ansible executes a deterministic pipeline from your control node (`SaltMaster`) to your managed nodes:

1. **Configuration Resolution:** Locates `ansible.cfg` to determine default inventory paths, privilege escalation rules, SSH timeouts, and module paths.
2. **Inventory Parsing:** Reads your target hosts (`inventory.ini`), resolves group hierarchies (`[roboshop:children]`), and loads host/group variables.
3. **SSH Connection & Python Discovery:** Connects to target machines over SSH (unless `connection: local` is specified) and locates a compatible Python interpreter (`/usr/bin/python3.12`).
4. **Fact Gathering (`setup` module):** Unless disabled (`gather_facts: no`), Ansible pushes a Python script to inspect the remote OS, IP addresses, CPU, RAM, and disks, storing everything in `ansible_facts`.
5. **Task Execution & Module Push:** Compiles each task into a self-contained Python module, pushes it over SSH, executes it, cleans up the temporary file, and returns JSON status (`ok`, `changed`, `failed`, `skipped`).

---

## 2. Core CLI Commands & Pre-Flight Validation

Always validate playbooks before running them against live infrastructure to catch indentation errors or unintended task runs:

| Command | Purpose & Behavior |
| :--- | :--- |
| `ansible --version` | Displays Ansible core version, active `ansible.cfg` path, Python version, and collection directories. |
| `ansible-playbook --syntax-check <file.yml>` | Validates YAML formatting and Ansible keyword structure without connecting to any hosts. |
| `ansible-playbook --list-tasks <file.yml>` | Previews every play and task name in execution order along with attached tags. |
| `ansible-playbook --check <file.yml>` | **Dry-Run Mode:** Simulates execution and predicts what *would* change without modifying the target system. |
| `ansible-playbook -i inventory.ini <file.yml>` | Overrides the default `/etc/ansible/hosts` inventory with your custom project inventory file. |

---

## 3. Configuration Precedence (`ansible.cfg`)

Ansible loads only **one** configuration file per run—it does not merge settings across multiple `.cfg` files. It checks four locations in strict top-down priority:

1. **`ANSIBLE_CONFIG` Environment Variable (Highest):** Overrides everything for the current shell session.
   ```bash
   export ANSIBLE_CONFIG=/home/mani/Ansible/learn-ansible/ansible-mani/ansible.cfg
   # Revert back to standard search order:
   unset ANSIBLE_CONFIG
   ```
2. **Current Working Directory (`./ansible.cfg`):** Ideal for project-specific Git repositories where each repo needs custom settings.
3. **User Home Directory (`~/.ansible.cfg`):** Applies user-level defaults across all projects.
4. **Global System Default (`/etc/ansible/ansible.cfg`) (Lowest):** Fallback configuration when no local or environment overrides exist.

---

## 4. Inventory Design (`inventory.ini`)

An inventory defines *what* infrastructure Ansible manages and *how* it connects to those nodes.

### Anatomy of `inventory.ini`
```ini
# 1. Ungrouped hosts (must appear at the top before any [group] header)
172.12.123.13
172.12.123.15

# 2. Functional tier groups
[frontend]
172.31.20.30

[backend]
172.12.123.50
172.12.123.51

[database]
172.12.123.45
172.12.123.46

# 3. Parent group (Group of Groups) using :children
# Targeting 'roboshop' applies tasks to frontend, backend, and database together
[roboshop:children]
frontend
backend
database

# 4. Group Variables using :vars
# CRITICAL: INI files strictly require '=' (key=value), NOT YAML's ':'
[Salt]
192.168.109.128
SaltMaster

[Salt:vars]
ansible_python_interpreter=/usr/bin/python3.12
```

* **Why `ansible_python_interpreter` matters:** Modern Linux distributions often have multiple Python versions installed. Explicitly pinning `/usr/bin/python3.12` eliminates interpreter discovery warnings and guarantees consistent module behavior across upgrades.

---

## 5. Playbook Structure: Single vs. Multi-Play (`01` to `04`)

A **Playbook** is a YAML file containing a list of one or more **Plays**. Each Play maps a group of target `hosts` to a sequence of `tasks`.

* **Local Execution (`01-playbook.yml`, `02-playbook.yml`):**
  ```yaml
  ---
  - name: ping demo
    hosts: localhost
    connection: local    # Bypasses SSH; runs directly on the control node
    tasks:
      - name: ping localhost
        ping:            # Tests Python execution capability (not ICMP network ping)
  ```
* **Multi-Play Architecture (`03-multiplaybook.yml`):**
  You can chain multiple plays in a single file. This is essential in real-world deployments where **Play 1** configures database servers (`hosts: database`), **Play 2** deploys backend APIs (`hosts: backend`), and **Play 3** configures reverse proxies (`hosts: frontend`).
* **Cross-Platform Service Management (`04-nginix.yml`):**
  * `package:` abstracts underlying OS package managers (`dnf`, `yum`, `apt`) for basic install/remove operations.
  * `service:` manages systemd/init daemons. Always pair `state: started` (starts now) with `enabled: yes` (starts automatically on reboot).

---

## 6. Privilege Escalation (`become`) & SSH Authentication

Administrative tasks (installing packages, modifying `/etc/passwd`, managing `systemd` units) fail unless both **playbook escalation** and **runtime credentials** are provided.

### How They Work Together
1. **Inside the Playbook:** Add `become: yes` (at the play level or individual task level) to tell Ansible to switch to `root` via `sudo`.
2. **At Runtime (CLI):** Supply the SSH login and `sudo` password using one of three methods:

* **Method 1 — Long flag prompt (Interactive & Secure):**
  ```bash
  ansible-playbook -i inventory.ini -e ansible_user=mani -e ansible_password='<ssh_pass>' --ask-become-pass 04-nginix.yml
  ```
* **Method 2 — Short flag `-K` prompt (Standard CLI workflow):**
  ```bash
  ansible-playbook -i inventory.ini -e ansible_user=mani -e ansible_password='<ssh_pass>' -K 04-nginix.yml
  ```
* **Method 3 — Non-interactive extra-vars (`-e`):**
  ```bash
  ansible-playbook -i inventory.ini -e ansible_user=mani -e ansible_password='<ssh_pass>' -e ansible_become_password='<sudo_pass>' 04-nginix.yml
  ```
  > **Security Tip:** Avoid passing plain-text passwords in CLI history in production; prefer SSH key pairs + `ansible-vault` or passwordless `sudo` for automation service accounts.

---

## 7. Data Types & Jinja2 Templating (`13-datatypes.yml`)

Ansible variables are parsed by YAML and evaluated by the Jinja2 templating engine. Understanding types prevents subtle logic bugs:

| Data Type | YAML Syntax Example | How Ansible Uses It |
| :--- | :--- | :--- |
| **String** | `"Netskope"` or `'Ansible'` | Text values; quote strings containing special characters or colons. |
| **Integer** | `95` | Whole numbers used in arithmetic or port definitions. |
| **Float** | `8.5` | Decimal numbers. |
| **Boolean** | `true` / `false` (or `yes` / `no`) | Feature flags, `become: yes`, `gather_facts: no`. |
| **List** | `["Ansible", "Salt", "Terraform"]` | Ordered collections iterated over with `loop`. |
| **Dictionary** | `{"city": "Santa Clara", "Stay": 5}` | Key-value maps accessed via dot notation (`dict.city`) or bracket notation (`dict['city']`). |
| **Null** | `null` (rendered as `None` in Python) | Empty/unset values. |

---

## 8. Variables: Sources & Precedence Hierarchy (`05` to `11`)

### 5 Ways to Supply Variables
1. **Inline Play `vars` (`05-variables.yml`):** Hardcoded inside the play for self-contained logic.
2. **External `vars_files` (`07-varsfromfiles.yml`):** Separates configuration data (`details.yml`) from playbook logic so playbooks stay clean and reusable.
3. **Interactive `vars_prompt` (`08-vars.yml`):** Pauses execution to collect user input. Use `private: yes` (default) for secrets/passwords and `private: no` for visible text.
4. **Inventory `[group:vars]` (`09-varsininventory.yml`):** Binds environment-specific values (like dev vs. prod IPs) directly to host groups.
5. **CLI Extra Vars `-e` (`10-vars.yml`):** Injects dynamic overrides from CI/CD pipelines (Jenkins, GitHub Actions) or terminal commands.

### Variable Precedence Order (Highest to Lowest)
If `my_name` is defined in multiple places simultaneously, the **higher-ranked source always wins** (`11-vars.yml`):

1. **Command-line values (`-e` / `--extra-vars`)** — *Unbeatable override*
2. **Task-level `vars`** — *Verified in `13-datatypes.yml`: overrides play-level `vars`*
3. **Block-level `vars`**
4. **Role and `include_vars`**
5. **Play-level `vars`**
6. **Play `vars_files`**
7. **Play `vars_prompt`**
8. **Inventory `host_vars` and `group_vars`**
9. **Ansible Facts (`ansible_facts`)**
10. **Registered variables (`register`)**
11. **`set_fact` variables**
12. **Role `defaults/main.yml`** — *Lowest priority baseline*

---

## 9. Facts Gathering (`12-gather_facts.yml`)

When `gather_facts: yes` is active, Ansible populates the `ansible_facts` dictionary with live system telemetry:

* `ansible_facts['os_family']` $\rightarrow$ `"RedHat"`, `"Debian"`, `"Suse"`
* `ansible_facts['default_ipv4']['address']` $\rightarrow$ Primary IPv4 address (`192.168.109.128`)
* `ansible_facts['processor_cores']` $\rightarrow$ CPU core count
* `ansible_facts['memfree_mb']` $\rightarrow$ Available RAM in MB
* `ansible_facts['architecture']` $\rightarrow$ `x86_64` or `aarch64`
* `ansible_facts['hostname']` $\rightarrow$ Short system hostname (`SaltMaster`)

> **Performance Tip:** Set `gather_facts: no` when running utility playbooks (like `13-datatypes.yml` or `18-filters.yml`) that do not inspect system properties—this cuts execution time significantly.

---

## 10. Conditionals (`when`) & Loops (`loop`)

### Conditionals (`14-conditions.yml`, `15-conditions.yml`)
The `when` clause evaluates a Jinja2 expression (note: do **not** wrap `when` expressions in `{{ }}`). If the condition evaluates to `false`, the task is marked `skipping`.

* **Critical Gotcha with `vars_prompt`:** Input from `vars_prompt` is always stored as a **String**. Performing modulo math (`number % 2`) on a string throws a type error unless you cast it with `| int`:
  ```yaml
  when: (number | int) % 2 == 0
  ```
* **Multi-OS Automation Pattern (`15-conditions.yml`):**
  ```yaml
  - name: Install nginx on Debian/Ubuntu
    apt:
      name: nginx
      state: present
    when: ansible_facts['os_family'] == "Debian"

  - name: Install nginx on RHEL/Rocky/CentOS
    dnf:
      name: nginx
      state: present
    when: ansible_facts['os_family'] == "RedHat"
  ```

### Loops (`16-loops.yml`)
Instead of writing repetitive `dnf` tasks, pass a list of dictionaries to `loop`. Ansible iterates through each entry and exposes it via the built-in `item` variable:

```yaml
vars:
  packages:
    - {"name": "git", "state": "present"}
    - {"name": "curl", "state": "present"}
    - {"name": "nginx", "state": "absent"}

tasks:
  - name: Manage packages dynamically
    dnf:
      name: "{{ item.name }}"
      state: "{{ item.state }}"
    loop: "{{ packages }}"
    when: ansible_facts['os_family'] == "RedHat"
```

---

## 11. Jinja2 Filters (`17-filters.yml`, `18-filters.yml`)

Filters transform variables inline using the pipe (`|`) operator. You can also chain multiple filters sequentially from left to right.

| Filter | Transformation | Lab Example | Result |
| :--- | :--- | :--- | :--- |
| `lower` | Lowercase conversion | `{{ ansible_facts['os_family'] \| lower }}` | `"redhat"` |
| `upper` | Uppercase conversion | `{{ ansible_facts['os_family'] \| upper }}` | `"REDHAT"` |
| `title` | Capitalizes each word | `{{ ansible_facts['os_family'] \| title }}` | `"Redhat"` |
| `default('val')` | Provides fallback if undefined | `{{ my_var \| default('mani_RHEL') }}` | `"mani_RHEL"` |
| `split(' ')` | Converts string into a list | `{{ "Mani Kumar" \| split(' ') }}` | `['Mani', 'Kumar']` |
| `trim` | Removes leading/trailing spaces | `{{ my_name \| trim }}` | `"Mani Kumar"` |

---

## 12. Error Handling, `register`, & Idempotency (`19` to `21`)

**Idempotency** is the core principle of configuration management: *applying the same playbook 1 time or 100 times must leave the system in the exact same desired state without failing or creating duplicates.*

### How Your `21-idempotency.yml` Pattern Works
1. **Probe State Safely (`register` + `ignore_errors: yes`):**
   Running `id roboshop` returns exit code `rc=0` if the user exists and `rc=1` (failure) if the user does not exist. Setting `ignore_errors: yes` prevents Ansible from aborting on `rc=1`, while `register: user_check` captures the return code.
2. **Act Only When Needed (`when: user_check.rc != 0`):**
   The `user` module executes *only* if the user is missing (`changed: [localhost]`). On the second run, `user_check.rc` is `0`, so the creation task is cleanly `skipped`.

```yaml
- name: Check if user exists
  command: id {{ person_name }}
  register: user_check
  ignore_errors: yes

- name: Create user only if missing
  user:
    name: "{{ person_name }}"
    state: present
  when: user_check.rc != 0

- name: Display existing user details
  debug:
    msg: "User check result: {{ user_check.stdout }}"
  when: user_check.rc == 0
```

---

## 13. `command` vs. `shell` Module (`22-shellvCommand.yml`)

Both modules execute ad-hoc Linux commands, but they differ fundamentally in how arguments are processed:

| Aspect | `command` Module | `shell` Module |
| :--- | :--- | :--- |
| **How it runs** | Executes the binary directly (`execve`) without invoking `/bin/sh`. | Wraps the command inside `/bin/sh -c "..."`. |
| **Pipes & Operators (`\|`, `>`, `<`, `;`, `&&`, `$VAR`)** | **Fails:** Treats `\|`, `grep`, and `roboshop` as literal filename arguments passed to `cat`. | **Works:** Interprets pipes, redirections, wildcards, and shell environment variables normally. |
| **When to use** | Default choice for simple single-binary execution (e.g., `id mani`, `uptime`) because it is immune to shell injection. | Use only when you genuinely need pipes (`cat /etc/passwd \| grep roboshop`), redirects, or compound shell expressions. |

---

## 14. Lab Troubleshooting Reference Matrix

Every real-world error encountered during your labs and its exact root-cause fix:

| Error Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| `skipping: no hosts matched` | Forgot `-i inventory.ini` (Ansible checked empty `/etc/ansible/hosts`) or used `hosts: local` instead of `hosts: localhost`. | Pass `-i inventory.ini` on the CLI and verify the `hosts:` pattern matches an inventory group or host. |
| `Failed to connect to the host via ssh: Permission denied` | Target host rejected unauthenticated SSH connection. | Pass `-e ansible_user=mani -e ansible_password=...` or configure Ed25519 SSH key-based auth. |
| `Missing sudo password` | Playbook has `become: yes`, but CLI lacked `-K` or `ansible_become_password`. | Add `-K` to the `ansible-playbook` command. |
| `useradd: Permission denied` / `SQLite error: readonly database` | Passed `-K` on the CLI, but forgot `become: yes` inside the playbook. | Add `become: yes` at the play or task level. |
| `'my_name' is undefined` (with INI inventory) | Defined variables directly under `[local]` or used `:` instead of `=`. | Place variables under a dedicated `[local:vars]` header using `key=value` syntax. |
| `object of type 'dict' has no attribute 'os_family'` | Evaluated `ansible_facts['os_family']` while `gather_facts: no` was set. | Change `gather_facts: yes` so the `setup` module populates `ansible_facts`. |
| `'package' is undefined` inside a loop | Referenced `{{ package.name }}` instead of Ansible's built-in loop variable. | Use `{{ item.name }}` (or define `loop_control: { loop_var: package }`). |