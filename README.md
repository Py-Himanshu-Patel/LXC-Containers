# LXC-Containers
A Setup and User guide to LXC Linux containers - Light weight VM

```
Heavy       --- >       Light  
Virtual Machine > LXC Machine Containers > Application Containers
```

### Virtual Machine
Bare Metal -> Linux OS -> Hypervisor -> Guest OS

### LXC Machine Containers
Bare Metal -> Linux OS -> LXC/LXD API's -> Guest OS

### Application Containers
Bare Metal -> Linux OS -> Container Runtime -> Application Binary

## Memory in VM vs LXC Containers
In case of Host Machine have 16 GB of memory.  
In case of VM, suppose there are two VM we made each of 4 GB then the 8GB of space is occupied by 2 VM and Host can't use that when the VM's are up. Also the Guest VM can't exceed the limit of 4GB.

While in case of LXC containers they share the same hardware as Host. Thus even if we start 2 LXC containers they can use up all 16GB of RAM. Though we can put a limit on LXC memory limit. This comes with a flexibility that Host machine do not lose any fixed partition of RAM.

## LXC Containers - Install and Init
```bash
$ sudo apt-get install lxc
$ which lxc
/snap/bin/lxc
$ lxc version
Client version: 5.12
Server version: 5.12
```
```bash
$ sudo systemctl status lxc
● lxc.service - LXC Container Initialization and Autoboot Code
     Loaded: loaded (/lib/systemd/system/lxc.service; enabled; vendor preset: enabled)
     Active: active (exited) since Mon 2023-03-27 11:16:37 IST; 1 week 5 days ago
       Docs: man:lxc-autostart
             man:lxc
   Main PID: 61591 (code=exited, status=0/SUCCESS)
        CPU: 15ms

Mar 27 11:16:37 workstation systemd[1]: Starting LXC Container Initialization and Autoboot Code...
Mar 27 11:16:37 workstation systemd[1]: Finished LXC Container Initialization and Autoboot Code.
```

To run lxc command without sudo add current user to lxd group
```bash
$ sudo gpasswd -a $USER lxd # adding user $USER to lxd group
$ getent group lxd  # see user in lxd group
lxd:x:135:lenovo
```
Init the LXC environment
```bash
$ lxd init
```

## LXC Commands

- Check the storage location for images of lxc containers
```bash
$ lxc storage list
+---------+--------+--------------------------------------------+-------------+---------+---------+
|  NAME   | DRIVER |                   SOURCE                   | DESCRIPTION | USED BY |  STATE  |
+---------+--------+--------------------------------------------+-------------+---------+---------+
| default | zfs    | /var/snap/lxd/common/lxd/disks/default.img |             | 4       | CREATED |
+---------+--------+--------------------------------------------+-------------+---------+---------+
```

- List all images we have
```bash
$ lxc image list
+-------+--------------+--------+---------------------------------------------+--------------+-----------+----------+-----------------------------+
| ALIAS | FINGERPRINT  | PUBLIC |                 DESCRIPTION                 | ARCHITECTURE |   TYPE    |   SIZE   |         UPLOAD DATE         |
+-------+--------------+--------+---------------------------------------------+--------------+-----------+----------+-----------------------------+
|       | 72565f3fbae4 | no     | ubuntu 22.04 LTS amd64 (release) (20230302) | x86_64       | CONTAINER | 442.27MB | Apr 2, 2023 at 6:15am (UTC) |
+-------+--------------+--------+---------------------------------------------+--------------+-----------+----------+-----------------------------+
```

- Launch LXC Container
```bash
$ lxc launch ubuntu:20.04
# OR
$ lxc launch ubuntu:20.04 dev   # give a name to container
```

- Check the container running / list all containers
```bash
$ lxc ls
+----------+---------+------+------+-----------+-----------+
|   NAME   |  STATE  | IPV4 | IPV6 |   TYPE    | SNAPSHOTS |
+----------+---------+------+------+-----------+-----------+
| dev      | RUNNING |      |      | CONTAINER | 0         |
+----------+---------+------+------+-----------+-----------+
```

- Delete the container - Stop first and then delete
```bash
$ lxc stop dev
$ lxc delete dev
```

- Start/Restart container
```bash
$ lxc start dev
$ lxc restart dev
```

- Make a copy of running container
```bash
$ lxc copy dev dev2 # existing_cont_name new_cont_name
# in case the new instance is stopped then start it as
$ lxc start dev2
```

- Move inside lxc container
```bash
$ lxc exec dev bash
# after moving inside another container ping another container
```

## Setup SSH for LXC Continer and AgentForwarding

Get the IP of container
```
$ lxc ls
+------+---------+----------------------+-----------------------------------------------+-----------+-----------+
| NAME |  STATE  |         IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+------+---------+----------------------+-----------------------------------------------+-----------+-----------+
| dev  | RUNNING | 10.193.32.55 (eth0)  | fd42:c573:22ef:63f5::1 (lxdbr0)               | CONTAINER | 0         |
+------+---------+----------------------+-----------------------------------------------+-----------+-----------+
```
### PermitRootLogin enabled for SSH
```bash
$ lxc exec dev bash
root@dev:~# vi /etc/ssh/sshd_config
```
Update the config
```
PermitRootLogin yes
PasswordAuthentication yes
```

### Put the Public Key of Host in LXD Container
- Get the Public Key from host
```bash
$ cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDQ0bdaqJf9xRqDdX++PeUGkHACIJWNsMIkpWjRHwHMm/+0ewTamJoHXfsCAOrEZSMUOkxuwTZsQbQw== himanshu.patel@clarisights.com
```
- Paste this key in LXC Container in a new line in below file
```bash
root@dev:~# vi ~/.ssh/authorized_keys

root@dev:~# cat ~/.ssh/authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDQ0bdaqJf9xRqDdX++PeUGkHACIJWNsMIkpWjRHwHMm/+0ewTamJoHXfsCAOrEZSMUOkxuwTZsQbQw== himanshu.patel@clarisights.com
```

### Save the SSH Config for LXD Container 
On Host file system save the SSH config with `ForwardAgent` as `yes`.
```
~/.ssh $ cat config 
Host dev
		HostName 10.193.32.55
		User 	root
		ForwardAgent yes
```
This agent forwarding allow LXD container to use SSH Agent of host machine. Like git pull of code and other auth work like LXC is host itself.

## Container info and profiles

- Get info of container
```bash
$ lxc info dev
Name: dev
Status: STOPPED
Type: container
Architecture: x86_64
Created: 2023/03/27 12:41 IST
Last Used: 2023/04/03 09:47 IST
```

```bash
$ lxc config show dev
architecture: x86_64
config:
  image.architecture: amd64
  image.description: ubuntu 20.04 LTS amd64 (release) (20230209)
  image.label: release
  image.os: ubuntu
  image.release: focal
  image.serial: "20230209"
  image.type: squashfs
  image.version: "20.04"
  volatile.last_state.power: STOPPED
  volatile.last_state.ready: "false"
  volatile.uuid: b4f615c0-9ec2-4f01-81ae-6ef9a5d98b80
devices: {}
ephemeral: false
profiles:
- default
stateful: false
description: ""
```

- Get all profile
```bash
$ lxc profile list  
# used by mean #of container using this profile
+---------+---------------------+---------+
|  NAME   |     DESCRIPTION     | USED BY |
+---------+---------------------+---------+
| default | Default LXD profile | 2       |
+---------+---------------------+---------+
```

- See Profile
```bash
$ lxc profile show default
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: default
used_by:
- /1.0/instances/dev
- /1.0/instances/practice
```

- Copy a profile 
```bash
# copy exising default profile to custom
$ lxc profile copy default custom

$ lxc profile list
+---------+---------------------+---------+
|  NAME   |     DESCRIPTION     | USED BY |
+---------+---------------------+---------+
| custom  | Default LXD profile | 0       |
+---------+---------------------+---------+
| default | Default LXD profile | 2       |
+---------+---------------------+---------+

# launch container using specific profile
$ lxc launch ubuntu:22.04 NewDev --profile custom
```

## Set Limits - Container level and Profile level

Current container use all 16GB of RAM. - inside of container shell
```bash
$ lxc exec dev bash
root@clarisights-dev:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          15708         143       15339          11         225       15565
Swap:             0           0           0
```

Update the limits - outside of container shell
```bash
$ lxc config set dev limits.memory 8GB
```
Check memory again
```bash
# free -m
              total        used        free      shared  buff/cache   available
Mem:           7629         169        7175          19         284        7460
Swap:             0           0           0
```

- Apply this limit on profile level then container level
```bash
$ lxc profile edit custom
# existing profile data
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: custom
used_by: []

# updated data - to apply limit on profile level
config: 
  limits.memory: 8GB
  limits.cpu: 1
description: Default LXD profile
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: custom
used_by: []
```

Don't forget to use profile flag in order to use updated profile
```bash
$ lxc launch ubuntu:22.04 NewDev --profile custom
```

## Snapshot, Restore and File Operations
- Send file from Host to Container
```bash
$ lxc file push data.txt dev/home/
# file_name.txt container_name/target_location/
```
- Pull file from container
```bash
$ lxc file pull dev/home/data.txt .
```

- Take a snapshot of a container
```bash
# container_name snap_name
$ lxc snapshot dev snap1
```
- Restore container from snapshot
```bash
# container_name snap_name
$ lxc restore dev snap1
```

## Nested Continers
- Move inside the continer and start lxd process there if not running already.
```bash
# start lxd container cluster
$ lxc init
```
- Set the container priviliged for nesting other containers
```bash
$ lxc config set dev security.privileged true
$ lxc config set dev security.nesting true
```
OR Update the profile as ealier
```
config: 
  limits.memory: 8GB
  limits.cpu: 1
  security.privileged: true
  security.nesting: true
```
Inside the `dev` container
```bash
# lxc launch ubuntu:22.04 nested-dev
Creating nested-dev
Starting nested-dev                           
:~# lxc ls

+------------+---------+---------------------+----------------------------------------------+-----------+-----------+
|    NAME    |  STATE  |        IPV4         |                     IPV6                     |   TYPE    | SNAPSHOTS |
+------------+---------+---------------------+----------------------------------------------+-----------+-----------+
| nested-dev | RUNNING | 10.155.67.52 (eth0) | fd42:c573:22ef:63f5:216:3eff:fe3b:bf9 (eth0) | CONTAINER | 0         |
+------------+---------+---------------------+----------------------------------------------+-----------+-----------+
```
Outside container/ on Host
```bash
$ lxc ls
+----------+---------+----------------------+-----------------------------------------------+-----------+-----------+
|   NAME   |  STATE  |         IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+----------+---------+----------------------+-----------------------------------------------+-----------+-----------+
| dev      | RUNNING | 10.193.32.55 (eth0)  | fd42:c573:22ef:63f5::1 (lxdbr0)               | CONTAINER | 0         |
|          |         | 10.155.67.1 (lxdbr0) | fd42:5921:cdd2:76cd:216:3eff:fece:ee68 (eth0) |           |           |
+----------+---------+----------------------+-----------------------------------------------+-----------+-----------+
```
