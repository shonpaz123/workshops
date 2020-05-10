# Exercise 4 - Network Configuration with Jinja Templates, Conditionals, Loops and Handlers

## Table of Contents

- [Objective](#objective)
- [Guide](#guide)
- [Takeaways](#takeaways)
- [Solution](#solution)

# Objective

Demonstration templating a network configuration and pushing it a device

- Use and understand group [variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html) to store the IP addresses we want.
- Use the [Jinja2 template lookup plugin](https://docs.ansible.com/ansible/latest/plugins/lookup.html)
- Demonstrate use of the network automation [cli_config module](https://docs.ansible.com/ansible/latest/modules/cli_config_module.html)

# Guide

#### Step 1

This step will cover creating Ansible variables for use in an Ansible Playbook. This exercise will use the following IP address schema for loopbacks addresses on rtr1 and rtr2:

Device  | Loopback100 IP |
------------ | ------------- |
rtr1  | 192.168.100.1/32 |
rtr2  | 192.168.100.2/32 |

Variable information can be stored in host_vars and group_vars.  For this exercise create a folder named `group_vars`:

```bash
[student1@ansible network-workshop]$ mkdir -p ~/networking-workshop/group_vars
```

Now create a file in this directory name `all.yml` using your text editor of choice.  Both vim and nano are installed on the control node.

```
[student1@ansible network-workshop]$ nano group_vars/all.yml
```

The interface and IP address information above must be stored as variables so that the Ansible playbook can use it. Start by making a simple YAML dictionary that stores the table listed above. Use a top level variable (e.g. `nodes`) so that a lookup can be performed based on the `inventory_hostname`:

```yaml
nodes:
  rtr1:
    Loopback100: "192.168.100.1"
  rtr2:
    Loopback100: "192.168.100.2"
```

Copy the YAML dictionary we created above into the group_vars/all.yml file and save the file.  

>All devices are part of the group **all** by default.  If we create a group named **cisco** only network devices belonging to that group would be able to access those variables.

#### Step 2

Create a new template file named `template.j2`:

```
[student1@ansible network-workshop]$ nano template.j2
```

Copy the following into the template.j2 file:

<!-- {% raw %} -->
```yaml
{% for interface,ip in nodes[inventory_hostname].items() %}
interface {{interface}}
  ip address {{ip}} 255.255.255.255
{% endfor %}
```
<!-- {% endraw %} -->


Save the file.

#### Step 3

This step will explain and elaborate on each part of the newly created template.j2 file.

<!-- {% raw %} -->
```yaml
{% for interface,ip in nodes[inventory_hostname].items() %}
```
<!-- {% endraw %} -->

<!-- {% raw %} -->
- Pieces of code in a Jinja template are escaped with `{%` and `%}`.  The `interface,ip` breaks down the dictionary into a key named `interface` and a value named `ip`.
<!-- {% endraw %} -->

- The `nodes[inventory_hostname]` does a dictionary lookup in the `group_vars/all.yml` file.  The **inventory_hostname** is the name of the hostname as configured in Ansible's inventory host file.  When the playbook is executed against `rtr1` inventory_hostname will be `rtr1`, when the playbook is executed against `rtr2`, the inventory_hostname will be `rtr2` and so forth.  

>The inventory_hostname variable is considered a [magic variable](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#magic-variables-and-how-to-access-information-about-other-hosts) which is automatically provided.  

- The keyword `items()` returns a list of dictionaries.  In this case the dictionary's key is the interface name (e.g. Loopback100) and the value is an IP address (e.g. 192.168.100.1)

<!-- {% raw %} -->
```yaml
interface {{interface}}
  ip address {{ip}} 255.255.255.255
```
<!-- {% endraw %} -->

- Variables are rendered with the curly braces like this: `{{ variable_here }}`  In this case the variable name key and value only exist in the context of the loop.  Outside of the loop those two variables don't exist.  Each iteration will re-assign the variable name to new values based on what we have in our variables.

Finally:
<!-- {% raw %} -->
```
{% endfor %}
```
<!-- {% endraw %} -->

- In Jinja we need to specify the end of the loop.

#### Step 4

Create the Ansible Playbook config.yml:

```bash
[student1@ansible network-workshop]$ nano config.yml
```

Copy the following Ansible Playbook to the config.yml file:

<!-- {% raw %} -->
```
---
- name: configure network devices
  hosts: rtr1,rtr2
  gather_facts: false
  tasks:
    - name: configure device with config
      cli_config:
        config: "{{ lookup('template', 'template.j2') }}"
```
<!-- {% endraw %} -->

- This Ansible Playbook has one task named *configure device with config*
- The **cli_config** module is vendor agnostic.  This module will work identically for an Arista, Cisco and Juniper device.  This module only works with the **network_cli** connection plugin.
- The cli_config module only requires one parameter, in this case **config** which can point to a flat file, or in this case uses the lookup plugin.  For a list of all available lookup plugins [visit the documentation](https://docs.ansible.com/ansible/latest/plugins/lookup.html)  
- Using the template lookup plugin requires two parameters, the plugin type *template* and the corresponding template name *template.j2*.

#### Step 5

Execute the Ansible Playbook:

```bash
[student1@ansible network-workshop]$ ansible-playbook config.yml
```

The output should look as follows.

```
[student1@ansible ~]$ ansible-playbook config.yml

PLAY [rtr1,rtr2] ********************************************************************************

TASK [configure device with config] ********************************************************************************
changed: [rtr1]
changed: [rtr2]

PLAY RECAP ********************************************************************************
rtr1                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
rtr2                       : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

#### Step 6

Use the command `show ip int br` to verify the IP addresses have been confirmed on the network devices.

```
[student1@ansible network-workshop]$ ssh rtr1

rtr1#show ip int br | include Loopback100
Loopback100            192.168.100.1   YES manual up                    up
```

#### Step 7 

Add the following task into your `config.yaml` playbook, which will fetch the MAC address for every interface configured in `ansible_net_interfaces` gathered by the facts: 

```bash 
    - name: display network interfraces 
      debug: 
        msg: "MAC for interface {{ item.key }} is {{ item.value['macaddress'] }} for host {{ ansible_net_hostname }}"
      with_dict: "{{ ansible_net_interfaces }}"
```

* **with_dict** is a way of iterating through a dictionary. In our facts, there is a variable called `ansible_net_interfaces` which contains a a list of each host's interfaces and their properties. 
* we use the **debug** module in order to print those values into the screen, for every interface under `ansible_net_interfaces`, this task will fetch the MAC adress of that interface and the hostname.
* **{{ item }}** is a way of using the item the loop/dict are iterating on (similar to foreach). the key in the dictionary is the interface name, where the value holds another dictionary with the interfaces properties.

Save the `config.yaml` file, and run the `ansible rtr1 -m ios_facts -a 'gather_subset=interfaces'` to verify that you have understand the data structure of the `ansible_net_interfaces` dictionary. 

Execute the ansible playbook

```[student1@ansible network-workshop]$ ansible-playbook config.yaml```

#### Step 8 

does it work? why? 

After you have understood that you should have added the `ios_facts` module to the playbook, rerun the playbook to get the proper result: 

```bash 
[student1@ansible networking-workshop]$ ansible-playbook config.yml 

PLAY [configure network devices] *************************************************************************************************************************************************************

TASK [gather router facts] *******************************************************************************************************************************************************************
[WARNING]: default value for `gather_subset` will be changed to `min` from `!config` v2.11 onwards
ok: [rtr1]
ok: [rtr2]

TASK [display network interfraces] ***********************************************************************************************************************************************************
ok: [rtr1] => (item={'key': 'GigabitEthernet1', 'value': {'description': None, 'macaddress': '0a15.3b94.7953', 'mtu': 1500, 'bandwidth': 1000000, 'mediatype': 'Virtual', 'duplex': 'Full', 'lineprotocol': 'up', 'operstatus': 'up', 'type': 'CSR vNIC', 'ipv4': [{'address': '172.16.182.64', 'subnet': '16'}]}}) => 
  msg: MAC for interface GigabitEthernet1 is 0a15.3b94.7953 for host rtr1
ok: [rtr1] => (item={'key': 'Loopback0', 'value': {'description': None, 'macaddress': None, 'mtu': 1514, 'bandwidth': 8000000, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '192.168.1.1', 'subnet': '32'}]}}) => 
  msg: MAC for interface Loopback0 is  for host rtr1
ok: [rtr1] => (item={'key': 'Loopback100', 'value': {'description': None, 'macaddress': None, 'mtu': 1514, 'bandwidth': 8000000, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '192.168.100.1', 'subnet': '32'}]}}) => 
  msg: MAC for interface Loopback100 is  for host rtr1
ok: [rtr1] => (item={'key': 'Tunnel0', 'value': {'description': None, 'macaddress': None, 'mtu': 9976, 'bandwidth': 100, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '10.100.100.1', 'subnet': '24'}]}}) => 
  msg: MAC for interface Tunnel0 is  for host rtr1
ok: [rtr1] => (item={'key': 'Tunnel1', 'value': {'description': None, 'macaddress': None, 'mtu': 9976, 'bandwidth': 100, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '10.200.200.1', 'subnet': '24'}]}}) => 
  msg: MAC for interface Tunnel1 is  for host rtr1
ok: [rtr1] => (item={'key': 'VirtualPortGroup0', 'value': {'description': None, 'macaddress': '001e.bdc2.b8bd', 'mtu': 1500, 'bandwidth': 750000, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': 'Virtual Port Group', 'ipv4': [{'address': '192.168.35.101', 'subnet': '24'}]}}) => 
  msg: MAC for interface VirtualPortGroup0 is 001e.bdc2.b8bd for host rtr1
ok: [rtr2] => (item={'key': 'GigabitEthernet1', 'value': {'description': None, 'macaddress': '0ad8.d523.8d79', 'mtu': 1500, 'bandwidth': 1000000, 'mediatype': 'Virtual', 'duplex': 'Full', 'lineprotocol': 'up', 'operstatus': 'up', 'type': 'CSR vNIC', 'ipv4': [{'address': '172.17.89.80', 'subnet': '16'}]}}) => 
  msg: MAC for interface GigabitEthernet1 is 0ad8.d523.8d79 for host rtr2
ok: [rtr2] => (item={'key': 'Loopback0', 'value': {'description': None, 'macaddress': None, 'mtu': 1514, 'bandwidth': 8000000, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '192.168.2.2', 'subnet': '32'}]}}) => 
  msg: MAC for interface Loopback0 is  for host rtr2
ok: [rtr2] => (item={'key': 'Loopback100', 'value': {'description': None, 'macaddress': None, 'mtu': 1514, 'bandwidth': 8000000, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '192.168.100.2', 'subnet': '32'}]}}) => 
  msg: MAC for interface Loopback100 is  for host rtr2
ok: [rtr2] => (item={'key': 'Tunnel0', 'value': {'description': None, 'macaddress': None, 'mtu': 9976, 'bandwidth': 100, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '10.101.101.2', 'subnet': '24'}]}}) => 
  msg: MAC for interface Tunnel0 is  for host rtr2
ok: [rtr2] => (item={'key': 'Tunnel1', 'value': {'description': None, 'macaddress': None, 'mtu': 9976, 'bandwidth': 100, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': None, 'ipv4': [{'address': '10.200.200.2', 'subnet': '24'}]}}) => 
  msg: MAC for interface Tunnel1 is  for host rtr2
ok: [rtr2] => (item={'key': 'VirtualPortGroup0', 'value': {'description': None, 'macaddress': '001e.f699.9ebd', 'mtu': 1500, 'bandwidth': 750000, 'mediatype': None, 'duplex': None, 'lineprotocol': 'up', 'operstatus': 'up', 'type': 'Virtual Port Group', 'ipv4': [{'address': '192.168.35.101', 'subnet': '24'}]}}) => 
  msg: MAC for interface VirtualPortGroup0 is 001e.f699.9ebd for host rtr2

TASK [configure device with config] **********************************************************************************************************************************************************
ok: [rtr2]
ok: [rtr1]

PLAY RECAP ***********************************************************************************************************************************************************************************
rtr1                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
rtr2                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

* Notice that under `msg:` you could see that those values indeed were fetched from the `ios_facts` module a passed into our task. In the second line you could see the dict structure, where the interface name is the key, and the interface's properties are the value. 

#### Step 9 

Add the following section into our `configure device with config` task in the `config.yaml` playbook, to use conditionals and handlers: 

```bash 
    - name: configure device with config 
      cli_config:
        config: "{{ lookup('template', 'template.j2') }}"
      when: ansible_net_iostype == "IOS-XE"
      notify:
      - write ios device

  handlers:
    - name: write ios device
      cli_command:
        command: write
```

* **when** is used as a conditional saying that this task will run **only** when the `ansible_net_iostype` variable is equal to `IOS-XE`. It means basically that for a device not running this ostype, will not get the configuration from the Jinja template. 
* **notify** will throw a notification to a handler called `write ios device` that is decalred under the `handlers` section. It is basically a task that runs only when the previous task calls it, a `write` command will be passed into the router to save those configurations. 
* The **write** command will be executed **only** if the state of the calling task has chaged. in our case, the task will only run if an `IOS-XE` device configuration **has changed**. 

#### Step 10 

```bash 
[student1@ansible network-workshop]$ ansible-playbook config.yaml
``` 

Have the handler been executed? why? 

#### Step 11 

Change the `group_vars/all.yaml` file, so that the handler will be notified due to the fact the state has changed: 

```bash 
nodes:
  rtr1:
    Loopback100: "192.168.100.2"
  rtr2:
    Loopback100: "192.168.100.3"

```

#### Step 12 

Execute the playbook and verify that the handler has been executed: 

```bash 

[student1@ansible network-workshop]$ ansible-playbook config.yaml -v 

.
.
.

TASK [configure device with config] **********************************************************************************************************************************************************
changed: [rtr2] => changed=true
changed: [rtr1] => changed=true

RUNNING HANDLER [write ios device] ***********************************************************************************************************************************************************
ok: [rtr2] => changed=false 
  stdout: |-
    Building configuration...
    [OK]
  stdout_lines: <omitted>
ok: [rtr1] => changed=false 
  stdout: |-
    Building configuration...
    [OK]
  stdout_lines: <omitted>
```

#### Step 12 

Login to the router and verify the configuration has indeed changed: 

```
[student1@ansible network-workshop]$ ssh rtr1

rtr1#show ip int br | include Loopback100
Loopback100            192.168.100.2   YES manual up                    up
``` 

# Takeaways

- The [Jinja2 template lookup plugin](https://docs.ansible.com/ansible/latest/plugins/lookup.html) can allow us to template out a device configuration.
- The `*os_config` (e.g. ios_config) and cli_config modules can source a jinja2 template file, and push directly to a device.  If you want to just render a configuration locally on the control node, use the [template module](https://docs.ansible.com/ansible/latest/modules/template_module.html).
- Variables are mostly commonly stored in group_vars and host_vars.  This short example only used group_vars.
- Loops can be used to iterate through dictionaries and lists. 
- Conditionals can be used to filter out specific cases in which a given task will run 
- Handlers can be notified whenever it needs to run by another task.

# Solution

The finished Ansible Playbook is provided here for an answer key: [config.yml](config.yml).

The provided Ansible Jinja2 template is provided here: [template.j2](template.j2).

---

# Complete

You have completed lab exercise 4

[Click here to return to the Ansible Network Automation Workshop](../README.md)
