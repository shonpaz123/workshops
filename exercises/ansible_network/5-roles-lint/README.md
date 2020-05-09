
# Exercise 5: Ansible Roles, Ansible lint

## Table of Contents

- [Objective](#objective)
- [Guide](#guide)
- [Takeaways](#takeaways)
- [Solution](#solution)

# Objective

Demonstration use of Ansible roles on network infrastructure.

Ansible holds an entity called `role`, which is a set of directory structure which helps us in packaging the Ansible ecosystem (playbooks, vars, templates) into specific roles, where each role has different responsebility. 

This exercise will cover:
- Building an Ansible Playbook from scratch.
- Using [ansible roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html).
- Using the [ansible galaxy](https://galaxy.ansible.com/).
- Using [ansible lint](https://docs.ansible.com/ansible-lint/).

# Guide

#### Step 1 

Create a directory that will contain our role: 

```bash 
[student1@ansible networking-workshop]$ mkdir -p ~/network-role && cd ~/network-role
```

#### Step 2 

Create a the role using the ansible-galaxy command: 

```bash 
[student1@ansible networking-workshop]$ ansible-galaxy init network-role 
- Role network-role was created successfully
```

* **ansible-galaxy` command can create empty roles for us to develop, and can reuse other roles that are being stored in the ansible-galaxy repository.

Use `tree` command to see the directory structure of oue role: 

```
[student1@ansible networking-workshop]$ tree network-role/
network-role/
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

#### Step 3 

Copy the following content into `network-role/templates/template.j2`: 

```bash 
{% for interface,ip in nodes[inventory_hostname].items() %}
interface {{interface}}
  ip address {{ip}} 255.255.255.255
{% endfor %}
```

* This template will be used as part of our role. 

Copy the following content into ```network-role/tasks/main.yml```:

```bash 
- name: gather router facts
  ios_facts:

- name: display network interfraces 
  debug: 
    msg: "MAC for interface {{ item.key }} is {{ item.value['macaddress'] }} for host {{ ansible_net_hostname }}"
  with_dict: "{{ ansible_net_interfaces }}"

- name: configure device with config 
  cli_config:
    config: "{{ lookup('template', 'template.j2') }}"
  when: ansible_net_iostype == "IOS-XE"
  notify:
  - write ios device
```

* Notice that we have no handler configuration, that is because we will use it in our `handlers` directory in the next steps.
* Notice we have no **hosts**, this is because this configuration is decalred at the role level. 

Copy the following content into `network-role/handlers/main.yml`:  

```bash 
- name: write ios device
  cli_command:
    command: write
```

* This handler will be called by our role, at the execution time the role will look for it in the wanted directory. 

Copy the `group_vars` directory into our working directory: 

```bash 
[student1@ansible networking-workshop]$ cp -r ~/networking-workshop/group_vars .
```

#### Step 4 

Create a `site.yml` file, which will package all of our executed roles: 

```bash 
---
- hosts: rtr1,rtr2
  gather_facts: false
  roles:
  - network-role
```

#### Step 5 

Execute the `site.yml` plabook and verify we have our role working as expected: 

```bash 
ansible-playbook site.yml 
```

#### Step 6 

change the `group_vars/all.yaml` configuration, and verify the handler is being executed succesfully.

#### Step 7 

Execute the ansible playbook:

```bash 
[student1@ansible networking-workshop]$ ansible-plabook site.yml 
.
.
.
TASK [network-role : configure device with config] *******************************************************************************************************************************************
changed: [rtr2]
changed: [rtr1]

RUNNING HANDLER [network-role : write ios device] ********************************************************************************************************************************************
ok: [rtr1]
ok: [rtr2]
```

#### Step 8 

Verify that the configuration has changed in our routers.

## Bonus 

#### Step 9 

Install `ansible-lint` to test your code: 

```bash 
[student1@ansible networking-workshop]$ pip install -y ansible-lint 
```

#### Step 10 

Chage directory to your home folder and lint our created `network-role` to verify it's valid: 

```bash 
[student1@ansible networking-workshop]$ cd ~/network-role && ansible-lint site.yml
```

* **ansible-lint** will crawl through our role with a givan `site.yml` file, it will collect the role's name and will look for it in the directory. 

#### Step 11

Fix the syntax problems your'e having **by yourself**

Congratulations! you have created your first tested ansible role!

# Takeaways

-  We can use [ansible roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html) to  package out playbooks into one single entity, giving us the ability of developing playbooks for each role. 
- We can use [ansible galaxy](https://galaxy.ansible.com/). to create out own empty roles, and also use existing roles located in the `ansible-galaxy` repository. 
- We can use [ansible lint](https://docs.ansible.com/ansible-lint/) to crawl through an existing role to do some testing. 

# Solution

The finished Ansible Role is provided here for an answer key: [final role](network-role/).

---

# Complete

You have completed lab exercise 5

[Click here to return to the Ansible Network Automation Workshop](../README.md)
