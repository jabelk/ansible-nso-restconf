# Learn by Doing: Infra as Code with Ansible RESTCONF on NSO

NSO has a ton of features. This repository is the first of a series which will show a simple use case, along with a feature. The purpose is dual:

1. See NSO applied to a variety of configuration situations and see how versatile it is..
1. Learn something new and have an example to follow

This example is using:

**Interface Loopback Config**

and the features I am showcasing are:

**RESTCONF Ansible Module**

This example assumes a working knowledge of Ansible and NSO. 

## Installation

[Reserve the NSO Reservable Sandbox](https://devnetsandbox.cisco.com/RM/Diagram/Index/43964e62-a13c-4929-bde7-a2f68ad6b27c?diagramType=Topology)

If you need to revisit some of the NSO Basics, you can [start here](https://developer.cisco.com/learning/lab/learn-nso-the-easy-way/step/1). 

Use some type of file transfer program or [VS Code has remote SSH](https://code.visualstudio.com/docs/remote/ssh) (drag and drop the package into the packages directory)



### Set Up

Log into the **devbox** (IP: 10.10.20.50) and install the [RESTCONF Ansible collection](https://docs.ansible.com/ansible/latest/collections/ansible/netcommon/restconf_config_module.html):

```
ansible-galaxy collection install ansible.netcommon
```

Clone or copy this repo onto the Devbox (IP: 10.10.20.50) home directory. Change to that directory to use the playbooks.

```
$ git clone https://github.com/jabelk/ansible-nso-restconf.git
Cloning into 'ansible-nso-restconf'...
remote: Enumerating objects: 14, done.
remote: Counting objects: 100% (14/14), done.
remote: Compressing objects: 100% (11/11), done.
Unpacking objects: 100% (14/14), done.
remote: Total 14 (delta 0), reused 14 (delta 0), pack-reused 0
$ cd ansible-nso-restconf/
ansible-nso-restconf$
```

### Execution

The playbooks are split up into a `build` and `push` two-phase approach. This is meant to mimic the mindset of Infra as Code, where the config is generated, edited, reviewed, tested and then pushed as separate steps.

To run the playbooks, ensure you are in the directory with the playbooks in them and then use the `ansible-playbook` command:

```
ansible-playbook  pb_restconf_get_build.yaml
```

The playbook will run, generating the `vars/int_vars.yaml` file. That file is already present in the repo, so you can delete it and the `int_vars_to_push.yaml` files if you want to simulate creating the files from scratch. 

Next, if you deleted the `int_vars_to_push.yaml` file to simulate needing to make it yourself, you will need to copy and paste the Loopback YAML into a new `int_vars_to_push.yaml` file and edit it. 

The point is that you don't need to write YAML from scratch because the `int_vars.yaml` creates the YAML structure for you, and all you need to do is copy and paste the part you want to merge into the device. 

Next, to push the new config, run the next playbook
```
ansible-playbook pb_restconf_push.yaml
```

The config will be pushed and some output will be shown. 

## Explanation of Components

Even though there are already official [Ansible NSO Modules](https://github.com/CiscoDevNet/ansible-nso), there are good reasons why you might want to use the Ansible RESTCONF modules with NSO. 

The official NSO Ansible modules use the JSON-RPC API, which allows for fancy commit options, and more nerd nobs. The downside is that many network engineers are not familiar with JSON-RPC, but are familiar with REST and/or RESTCONF. This means troubleshooting and development might be easier if using the RESTCONF modules. Also, currently the JSON-RPC Ansible modules accept JSON and not YAML as an input payload, which adds another layer of complexity in translating files written and maintained in YAML into JSON before sending them off. 

Just as a point of clarification for those who do not already know, we are using the NSO RESTCONF API to configure the application, which then in turn uses the NSO NEDs to sculpt CLI commands to the device. We are not using RESTCONF to talk directly to devices. Since NSO conforms to the IETF standard way of responding to RESTCONF calls, we can simply reuse the RESTCONF Ansible modules to configure NSO. 

### Variables Used

We are using two files with variables (apart from the two `int_vars*` files we build during the playbook execution), `group_vars/all.yaml` and `inventory.yaml`

The group vars file allows us to define the credentials, and a few other relevant HTTP based settings to have our playbooks be clean and simple. 

```yaml
---
# Connectivity parameters
ansible_connection: "httpapi"
ansible_network_os: "restconf"
ansible_httpapi_use_ssl: true
ansible_httpapi_validate_certs: false
ansible_httpapi_port: 443
ansible_httpapi_restconf_root: "/restconf/data/"  # default
ansible_user: "developer"
ansible_password: "C1sco12345"
```

The inventory file simply has the IP address and inventory hostname. 

```yaml
---
all:
  children:
    nso_server:
      hosts:
        latest:
          ansible_host: "10.10.20.49" 
...
```

### Templates Used

I used the following template to and a Jinja filter to transform our JSON response from NSO into YAML:

```
{{ int_config | to_nice_yaml }}
```

### Build Playbook

The "build" playbook uses the RESTCONF GET module to get the interface config from `dist-rtr01` and store that in a `restconf_result` variable. We then extract out the `response` payload data and plug that into our Jinja template to transform it into YAML. 

```yaml

- name: BUILD CONFIG WITH RESTCONF GET AND NSO
  hosts: "nso_server"
  gather_facts: no
  tasks:

  - name: RESTCONF GET CONFIG FROM DIST-RTR01 FROM NSO 
    ansible.netcommon.restconf_get:
        path: "/tailf-ncs:devices/device=dist-rtr01/config/tailf-ned-cisco-ios:interface/"
    register: restconf_result
  
  - name: VIEW DEVICE INTERFACE CONFIG
    debug: var=restconf_result

  - name: SET FACT TO EXTRACT RESPONSE VALUE
    set_fact: 
      int_config: "{{ restconf_result.response }}"

  - name: USE JINJA2 TO TRANSFORM JSON OUTPUT TO YAML FILE
    template:
      lstrip_blocks: yes
      src: json2yaml.j2
      dest: vars/int_vars.yaml

# then create vars/int_vars_to_push.yaml file manually and copy and paste the Loopback YAML
# in there to have config to work with to push with next playbook
    # Loopback:
    # -   description: to
    #     ip:
    #         no-address:
    #             address: false
    #     name: '0'
    #     shutdown:
    #     - null

```

The assumption is, if this was run the first time, with no files in `vars/`, it would generate a YAML file with all the interface config. Then you could take the loopback config from the generated file:

```yaml
tailf-ned-cisco-ios:interface:
    GigabitEthernet:
    -   description: to port6.sandbox-backend
        ip:
            address:
                primary:
                    address: 10.10.20.175
                    mask: 255.255.255.0
        mop:
            sysid: false
            xenabled: false
        name: '1'
        negotiation:
            auto: true
        vrf:
            forwarding: Mgmt-intf
    -   description: L3 Link to core-rtr01
        ip:
            address:
                primary:
                    address: 172.16.252.21
                    mask: 255.255.255.252
        mop:
            sysid: false
            xenabled: false
        name: '2'
        negotiation:
            auto: true
    -   description: L3 Link to core-rtr02
        ip:
            address:
                primary:
                    address: 172.16.252.25
                    mask: 255.255.255.252
        mop:
            sysid: false
            xenabled: false
        name: '3'
        negotiation:
            auto: true
    -   description: L3 Link to dist-sw01
        ip:
            address:
                primary:
                    address: 172.16.252.2
                    mask: 255.255.255.252
        mop:
            sysid: false
            xenabled: false
        name: '4'
        negotiation:
            auto: true
    -   description: L3 Link to dist-sw02
        ip:
            address:
                primary:
                    address: 172.16.252.10
                    mask: 255.255.255.252
        mop:
            sysid: false
            xenabled: false
        name: '5'
        negotiation:
            auto: true
    -   description: L3 Link to dist-rtr02
        ip:
            address:
                primary:
                    address: 172.16.252.17
                    mask: 255.255.255.252
        mop:
            sysid: false
            xenabled: false
        name: '6'
        negotiation:
            auto: true
    Loopback:
    -   description: to
        ip:
            no-address:
                address: false
        name: '0'
        shutdown:
        - null
```

and edit it to add a description and new loopback number in `int_vars_to_push.yaml`:

```yaml
tailf-ned-cisco-ios:interface:
    Loopback:
    -   description: "DESCRIPTION BY ANSIBLE"
        ip:
            no-address:
                address: false
        name: '100'
        shutdown:
        - null
```

### Push Playbook

The push playbook assumes the `int_vars_to_push.yaml` file is ready to go. It loads in the variable file, then uses RESTCONF PATCH to `dist-rtr01` to push the new loopback config. There is then a task to print out the new interface list with the new loopback.

Finally, I added a task to delete the loopback, just to show how that would be done, and also so when the playbook was run multiple times for testing, it would always create the loopback. 

```yaml

- name: PUSH CONFIG WITH RESTCONF CONFIG AND NSO
  hosts: "nso_server"
  gather_facts: no
  tasks:

  - name: LOAD LOOPBACK CONFIG 
    include_vars:
      file: int_vars_to_push.yaml
      name: int_vars_file

  - name: ADD LOOPBACK 100 TO NSO AND DEVICE
    register: restconf_result
    ansible.netcommon.restconf_config:
        path: "/tailf-ncs:devices/device=dist-rtr01/config/tailf-ned-cisco-ios:interface"
        method: "patch"
        content: "{{ int_vars_file | to_json }}"

  - name: VERIFY NEW LOOPBACK IS PRESENT
    ansible.netcommon.restconf_get:
        path: "/tailf-ncs:devices/device=dist-rtr01/config/tailf-ned-cisco-ios:interface/"
    register: restconf_result

# used .get() to grab key to avoid expected token 'end of print statement', got ':' error  
  - name: VIEW DEVICE INTERFACE CONFIG
    debug: 
      var: "restconf_result.response.get('tailf-ned-cisco-ios:interface').Loopback"

  - name: DELETE LOOPBACK 100 FROM NSO AND DEVICE
    register: restconf_result
    ansible.netcommon.restconf_config:
        path: "/tailf-ncs:devices/device=dist-rtr01/config/tailf-ned-cisco-ios:interface/Loopback=100"
        method: "delete"
      

 
```


## Sample Output Build

```bash
(py3venv) [developer@devbox ansible-nso-restconf]$ ansible-playbook pb_restconf_get_build.yaml

PLAY [BUILD CONFIG WITH RESTCONF GET AND NSO] **************************************************************************************

TASK [RESTCONF GET CONFIG FROM DIST-RTR01 FROM NSO] ********************************************************************************
ok: [latest]

TASK [VIEW DEVICE INTERFACE CONFIG] ************************************************************************************************
ok: [latest] => {
    "restconf_result": {
        "changed": false,
        "failed": false,
        "response": {
            "tailf-ned-cisco-ios:interface": {
                "GigabitEthernet": [
                    {
                        "description": "to port6.sandbox-backend",
                        "ip": {
                            "address": {
                                "primary": {
                                    "address": "10.10.20.175",
                                    "mask": "255.255.255.0"
                                }
                            }
                        },
                        "mop": {
                            "sysid": false,
                            "xenabled": false
                        },
                        "name": "1",
                        "negotiation": {
                            "auto": true
                        },
                        "vrf": {
                            "forwarding": "Mgmt-intf"
                        }
                    },
                    {
                        "description": "L3 Link to core-rtr01",
                        "ip": {
                            "address": {
                                "primary": {
                                    "address": "172.16.252.21",
                                    "mask": "255.255.255.252"
                                }
                            }
                        },
                        "mop": {
                            "sysid": false,
                            "xenabled": false
                        },
                        "name": "2",
                        "negotiation": {
                            "auto": true
                        }
                    },
                    {
                        "description": "L3 Link to core-rtr02",
                        "ip": {
                            "address": {
                                "primary": {
                                    "address": "172.16.252.25",
                                    "mask": "255.255.255.252"
                                }
                            }
                        },
                        "mop": {
                            "sysid": false,
                            "xenabled": false
                        },
                        "name": "3",
                        "negotiation": {
                            "auto": true
                        }
                    },
                    {
                        "description": "L3 Link to dist-sw01",
                        "ip": {
                            "address": {
                                "primary": {
                                    "address": "172.16.252.2",
                                    "mask": "255.255.255.252"
                                }
                            }
                        },
                        "mop": {
                            "sysid": false,
                            "xenabled": false
                        },
                        "name": "4",
                        "negotiation": {
                            "auto": true
                        }
                    },
                    {
                        "description": "L3 Link to dist-sw02",
                        "ip": {
                            "address": {
                                "primary": {
                                    "address": "172.16.252.10",
                                    "mask": "255.255.255.252"
                                }
                            }
                        },
                        "mop": {
                            "sysid": false,
                            "xenabled": false
                        },
                        "name": "5",
                        "negotiation": {
                            "auto": true
                        }
                    },
                    {
                        "description": "L3 Link to dist-rtr02",
                        "ip": {
                            "address": {
                                "primary": {
                                    "address": "172.16.252.17",
                                    "mask": "255.255.255.252"
                                }
                            }
                        },
                        "mop": {
                            "sysid": false,
                            "xenabled": false
                        },
                        "name": "6",
                        "negotiation": {
                            "auto": true
                        }
                    }
                ],
                "Loopback": [
                    {
                        "description": "to",
                        "ip": {
                            "no-address": {
                                "address": false
                            }
                        },
                        "name": "0",
                        "shutdown": [
                            null
                        ]
                    }
                ]
            }
        }
    }
}

TASK [SET FACT TO EXTRACT RESPONSE VALUE] ******************************************************************************************
ok: [latest]

TASK [USE JINJA2 TO TRANSFORM JSON OUTPUT TO YAML FILE] ****************************************************************************
ok: [latest]

PLAY RECAP *************************************************************************************************************************
latest                     : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

(py3venv) [developer@devbox ansible-nso-restconf]$
```

## Sample Output Push

```bash
(py3venv) [developer@devbox ansible-nso-restconf]$ ansible-playbook pb_restconf_push.yaml

PLAY [PUSH CONFIG WITH RESTCONF CONFIG AND NSO] ************************************************************************************

TASK [LOAD LOOPBACK CONFIG] ********************************************************************************************************
ok: [latest]

TASK [ADD LOOPBACK 100 TO NSO AND DEVICE] ******************************************************************************************
changed: [latest]

TASK [VERIFY NEW LOOPBACK IS PRESENT] **********************************************************************************************
ok: [latest]

TASK [VIEW DEVICE INTERFACE CONFIG] ************************************************************************************************
ok: [latest] => {
    "restconf_result.response.get('tailf-ned-cisco-ios:interface').Loopback": [
        {
            "description": "to",
            "ip": {
                "no-address": {
                    "address": false
                }
            },
            "name": "0",
            "shutdown": [
                null
            ]
        },
        {
            "description": "DESCRIPTION BY ANSIBLE",
            "ip": {
                "no-address": {
                    "address": false
                }
            },
            "name": "100",
            "shutdown": [
                null
            ]
        }
    ]
}

TASK [DELETE LOOPBACK 100 FROM NSO AND DEVICE] *************************************************************************************
changed: [latest]

PLAY RECAP *************************************************************************************************************************
latest                     : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

(py3venv) [developer@devbox ansible-nso-restconf]$
```