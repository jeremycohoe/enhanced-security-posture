# Enhanced Security Posture with Ansible and ZTP

There is a usecase where we want to apply some additional CLI's to the device to configure an enhanced security posture

You should configure the ACL according to network requirments, the IP's are just an example.

## CLI Configuration
```
! disable small servers
no service tcp-small-servers
no service udp-small-servers
! create ACL
 ip access-list 101 
! block IP
deny ip 99.99.99.99 any log
! allow mgmt network
permit ip 10.1.1.0 0.0.0.255 any
! Apply ACL to VTY lines:
line vty 0 32
access-class 101 in
```

## CLI + EEM 
In order to apply this CLI configuration an EEM Applet can be used 

```
conf t
alias exec secure_config event manager run secure_config

no event manager applet secure_config
event manager applet secure_config
 event none maxrun 300
 action 0001 cli command "enable"
 action 0002 cli command "conf t"
 action 0030 cli command "no service tcp-small-servers"
 action 0040 cli command "no service udp-small-servers"
 action 0050 cli command "ip access-list 101"
 action 0060 cli command "deny ip 99.99.99.99 any log"
 action 0061 cli command "permit ip any any"
 action 0070 cli command "line vty 0 32"
 action 0080 cli command "access-class 101 in"
end

! Call the alias to active the EEM applet:
secure_config
```


## Ansible

Our tooling of choice today is Ansible. The hosts file has bunch of devices that will be used to send the CLI to. This list is exported from Cisco DNA Center API

### ansible.cfg

```
[defaults]
inventory = ./hosts
host_key_checking = False
timeout = 6000
deprecation_warnings=False
```

### hosts
```
[VNC2]
jcohoe-c9300    ansible_host=10.85.134.65
jcohoe-c9300-2  ansible_host=10.85.134.70
jcohoe-c9200    ansible_host=10.85.134.72
jcohoe-c9500    ansible_host=10.85.134.84
vnc2-border1    ansible_host=10.85.134.92
vnc2-border2    ansible_host=10.85.134.94
vnc2-spine2     ansible_host=10.85.134.95
vnc2-leaf1      ansible_host=10.85.134.99
vnc2-leaf2      ansible_host=10.85.134.98
vnc2-leaf3      ansible_host=10.85.134.97
vnc2-leaf4      ansible_host=10.85.134.96
jcohoe-asr1000  ansible_host=10.85.134.89
jcohoe-isr4321  ansible_host=10.85.134.77
jcohoe-c9840    ansible_host=10.85.134.83
jcohoe-c9800l   ansible_host=10.85.134.78
jcohoe-console  ansible_host=10.85.134.79

[VNC2-switching]
jcohoe-c9300    ansible_host=10.85.134.65
jcohoe-c9300-2  ansible_host=10.85.134.70
jcohoe-c9200    ansible_host=10.85.134.72
jcohoe-c9500    ansible_host=10.85.134.84
vnc2-border1    ansible_host=10.85.134.92
vnc2-border2    ansible_host=10.85.134.94
vnc2-spine2     ansible_host=10.85.134.95
vnc2-leaf1      ansible_host=10.85.134.99
vnc2-leaf2      ansible_host=10.85.134.98
vnc2-leaf3      ansible_host=10.85.134.97
vnc2-leaf4      ansible_host=10.85.134.96

[VNC2-routing]
jcohoe-asr1000  ansible_host=10.85.134.89
jcohoe-isr4321  ansible_host=10.85.134.77
jcohoe-c9840    ansible_host=10.85.134.83
jcohoe-c9800l   ansible_host=10.85.134.78
jcohoe-console  ansible_host=10.85.134.79

[all:vars]
ansible_ssh_user=admin
ansible_ssh_pass=not_your_password
ansible_network_os=ios
```

### ansible.yaml
```
---
- name: Configure enhanced security posture
  hosts: VNC2
  gather_facts: no
  connection: network_cli

  tasks:
    - name: Configure Syslog and Remove legacy EEM applet
      ios_config:
        lines:
          - logging host 10.85.134.66 vrf Mgmt-vrf
          - no event manager applet catchall
        
    - name: Enable EEM applet for monitoring
      ios_config:
        lines:
          - event manager applet catchall
          - event cli pattern ".*" sync no skip no
          - action 1 syslog msg "$_cli_msg"
          - end

    - name: Remove legacy ACL if applied
      ios_config:
        lines:
          - no ip access-list extended 101
          - line vty 0 32
          - no access-class 101 in

    - name: Configure ACL
      ios_config:
        lines:
            - 11 permit ip 10.85.134.0 0.0.0.255 any
            - 12 deny ip host 99.99.99.99 any log
        parents: ip access-list extended 101
        before: no ip access-list extended 101
        match: exact

    - name: Apply ACL and disable small servers
      ios_config:
        lines:
          - no service tcp-small-servers
          - no service udp-small-servers
          - line vty 0 32
          - access-class 101 in 

    - name: Save config
      ios_command:
        commands:
          - wr mem
```

## ZTP
For new devices that are being deployed we can leverage the Zero Touch Provisioning featureset. 

### ztp.py
This config sample can be used in the larger ZTP Python configuration file. 

* First an EEM applet is defined that has the required CLI's set.
* Next the EEM applet is configured into the running configuration
* Then an alias is created so that the EEM applet can be executed easily
* Finally the "secure_config" CLI is called, the alias is used, and the EEM applet is run to configure the CLI's


```
# Enable secure_config EEM applet and script
print ("*** Configure secure_config examples on device... ***")
eem_secure_config_commands = ['no event manager applet secure_config',
                'event manager applet secure_config',
                'event none maxrun 300',
                'action 0001 cli command "enable"',
                'action 0002 cli command "conf t"',
                'action 0030 cli command "no service tcp-small-servers"',
                'action 0040 cli command "no service udp-small-servers"',
                'action 0050 cli command "ip access-list 101"',
                'action 0060 deny ip 99.99.99.99 any log”’,
                'action 0061 permit ip any any”’,
                'action 0070 cli command "line vty 0 32"',
                'action 0080 cli command "access-class 101 in"',
                'action 0090 cli command "end"',
]
results = configure(eem_secure_config_commands)
# Create alias secure_config to run the EEM applet with:
cli.configurep(["alias exec secure_config event manager run secure_config", "exit"])
cli_command = "secure_config"
cli.executep(cli_command)
``` 
 
Alternatley, the desired CLI's can be baked directly into the ZTP.py file without the EEM Applet and alias:

```
cli.configurep(["no service tcp-small-servers", "no service udp-small-servers"])
cli.configurep(["ip access-list 101", "deny ip 99.99.99.99 any log", "permit ip any any", "line vty 0 32", "access class 101 in", "end"])
```


 
