# Testbed Setup

This document describes the steps to setup the virtual switch based testbed and deploy a topology.

## Prepare testbed server

- Install Ubuntu 18.04 amd64 server. To setup a T0 topology, the server needs to have 10GB free memory.
- Install bridge utils
```
$ sudo apt-get install bridge-utils
```
- Setup internal management network.

```
$ sudo brctl addbr br1
$ sudo ifconfig br1 10.250.0.1/24
$ sudo ifconfig br1 up
```


- Download vEOS image from [arista](https://www.arista.com/en/support/software-download).
- Copy below image files to ```~/veos-vm/images``` on your testbed server.
   - ```Aboot-veos-serial-8.0.0.iso```
   - ```vEOS-lab-4.15.9M.vmdk```

## Setup docker registry for *PTF* docker

PTF docker is used to send and receive packets to test data plane. 

- Build PTF docker
```
$ git clone --recursive https://github.com/Azure/sonic-buildimage.git
$ make configure PLATFORM=generic
$ make target/docker-ptf.gz
```

- Download pre-built *docker-ptf* image from [here](https://sonic-jenkins.westus2.cloudapp.azure.com/job/broadcom/job/buildimage-brcm-all/lastSuccessfulBuild/artifact/target/docker-ptf-brcm.gz)
```
$ wget https://sonic-jenkins.westus2.cloudapp.azure.com/job/broadcom/job/buildimage-brcm-all/lastSuccessfulBuild/artifact/target/docker-ptf-brcm.gz
```

- Load *docker-ptf* image
```
$ docker load -i docker-ptf-brcm.gz
```

## Build or download *sonic-mgmt* docker image

ansible playbook in *sonic-mgmt* repo requires to setup ansible and various dependencies.
We have built a *sonic-mgmt* docker that installs all dependencies, and you can build 
the docker and run ansible playbook inside the docker.

- Build *sonic-mgmt* docker
```
$ git clone --recursive https://github.com/Azure/sonic-buildimage.git
$ make configure PLATFORM=generic
$ make target/docker-sonic-mgmt.gz
```

- Download pre-built *sonic-mgmt* image from [here](https://sonic-jenkins.westus2.cloudapp.azure.com/job/bldenv/job/docker-sonic-mgmt/lastSuccessfulBuild/artifact/target/docker-sonic-mgmt.gz).
```
$ wget https://sonic-jenkins.westus2.cloudapp.azure.com/job/bldenv/job/docker-sonic-mgmt/lastSuccessfulBuild/artifact/target/docker-sonic-mgmt.gz
```

- Load *sonic-mgmt* image
```
$ docker load -i docker-sonic-mgmt.gz
```

## Download sonic-vs image

- Download sonic-vs image from [here](https://sonic-jenkins.westus2.cloudapp.azure.com/job/vs/job/buildimage-vs-image/lastSuccessfulBuild/artifact/target/sonic-vs.img.gz)
```
$ wget https://sonic-jenkins.westus2.cloudapp.azure.com/job/vs/job/buildimage-vs-image/lastSuccessfulBuild/artifact/target/sonic-vs.img.gz
```

- unzip the image and move it into ```~/sonic-vm/images/```
```
$ gzip -d sonic-vs.img.gz
$ mkdir -p ~/sonic-vm/images
$ mv sonic-vs.img ~/sonic-vm/images
```

## Clone sonic-mgmt repo

```
$ git clone https://github.com/Azure/sonic-mgmt
```

### Modify login user name
```
lgh@gulv-vm2:/data/sonic/sonic-mgmt/ansible$ git diff
diff --git a/ansible/veos.vtb b/ansible/veos.vtb
index 4ea5a7a..4cfc448 100644
--- a/ansible/veos.vtb
+++ b/ansible/veos.vtb
@@ -1,5 +1,5 @@
[vm_host_1]
-STR-ACS-VSERV-01 ansible_host=172.17.0.1 ansible_user=use_own_value
+STR-ACS-VSERV-01 ansible_host=172.17.0.1 ansible_user=lgh

 [vm_host:children]
vm_host_1
```

## Run sonic-mgmt docker

```
$ docker run -v $PWD:/data -it docker-sonic-mgmt bash
```

From now on, all steps are running inside the *sonic-mgmt* docker.

### Setup public key to login into the linux host from sonic-mgmt docker

- Modify veos.vtb to use the user name to login linux host. Add public key to authorized\_keys for your user. 
Put the private key inside the sonic-mgmt docker container. Make sure you can login into box using 
```ssh yourusername@172.17.0.1``` without any password prompt inside the docker container.

-Create a ssh public key from docker sonic mgmt container and copy it to authorized_keys file in the server.
johnar@9cbdf85d4f63:~/sonic-mgmt/ansible$ cd ~
johnar@9cbdf85d4f63:~$ cd ~/.ssh
johnar@9cbdf85d4f63:~/.ssh$ ls
authorized_keys2  config  id_rsa  id_rsa.pub
johnar@9cbdf85d4f63:~/.ssh$ cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDe1SD7efOOtrI5xP6nMud7IOl4kkWniMHO/5Rl7AbmKfQ06puc/MKodd7qk5tvhnUAFO53i3QAstYURnyZPA+KY2VWe1WXUlI2hAfJdznMEw9XVkQQFf4P87pBxALdpf+ZJyxyaXD9xJl5XVlmIR3EaC43X+LNGcMS+O7tbmTu562CPZuUK3KMs5CLWTMKuDnld3uMFqaBsqdND2w9Qnt/ulFGn4fzuBiHKBGqSF5EXjx8XG5t5uoKzPb9fg6KnO2AV3rw8fnh81SaiQJ3v9AB+HVSPS/sYB0IaDvmcuH9YStDe4vhKWjOf7u1E0yFIVeHuvott7hUoXERMRiDodB9 johnar@9cbdf85d4f63

root@ubuntu:~/.ssh# cat authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDpJeV5t35DnvEpG4HgiCdHKGg5rMH+Ngm/utrPUxJ++9Zo5LyotBHt/gK/TmWEUjIbRFwJkjbJZ7c3T0Fkt5+RRlR67IDaPXvWj32RMt3B5jCV3NurYVR6BU1MOqndeptiSG6Z8Ckl09sCJkXpkUW0+iz6QeCekEwF6nq4M68LZFrIungIxY+S28R/Tu8ktQbmlq3UmOcLayFK+CDf6albfuVYIVZQ2nYMApMfkgyVW8G9uDT9jbnCheG/QUPUjgi1eaacbByvEhrlt4Jhhzqlu3nLWHJOqfjDPbmVbPgbY/4PcONqaODOsmFACv6/m7lsp691OKLL3BbeND/Aiqu7 johnar@4703be06bf1e
root@ubuntu:~/.ssh#


### Configure veos.vtb file 

-Modify veos.vtb file to use ansible user and ansible host ip address details.

```
[vm_host_1]
STR-ACS-VSERV-01 ansible_host=172.17.0.1 ansible_user=administrator

[vm_host:children]
vm_host_1

[vms_1]
VM0400 ansible_host=10.16.207.51
VM0401 ansible_host=10.16.207.52
VM0402 ansible_host=10.16.207.53
VM0403 ansible_host=10.16.207.54


[eos:children]
vms_1

## The groups below are helper to limit running playbooks to server_1, server_2 or server_3 only
[server_1:children]
vm_host_1
vms_1

[server_1:vars]
host_var_file=host_vars/STR-ACS-VSERV-01.yml

[servers:children]
server_1

[servers:vars]
topologies=['t1', 't1-lag', 't1-64-lag', 't0', 't0-16', 't0-56', 't0-52', 'ptf32', 'ptf64', 't0-64', 't0-64-32', 't0-116']

[sonic]
vlab-01 ansible_host=10.16.207.80 type=kvm hwsku=Force10-S6000
vlab-02 ansible_host=10.16.207.81 type=kvm hwsku=Force10-S6100
```

## Setup Arista VMs in the server

```
$ ./testbed-cli.sh -m veos.vtb start-vms server_1 password.txt
```
  - please note: Here "password.txt" is the ansible vault password file name/path. Ansible allows user use ansible vault to encrypt password files. By default, this shell script require a password file. If you are not using ansible vault, just create an empty file and pass the filename to the command line. The file name and location is created and maintained by user. 

Check that all VMs are up and running, and the passwd is ```123456```
```
$ ansible -m ping -i veos.vtb server_1 -u root -k
VM0102 | SUCCESS => {
        "changed": false, 
                "ping": "pong"
}
VM0101 | SUCCESS => {
        "changed": false, 
                "ping": "pong"
}
STR-ACS-VSERV-01 | SUCCESS => {
        "changed": false, 
                "ping": "pong"
}
VM0103 | SUCCESS => {
        "changed": false, 
                "ping": "pong"
}
VM0100 | SUCCESS => {
        "changed": false, 
                "ping": "pong"
}
```


## Deploy T0 topology

```
$ ./testbed-cli.sh -t vtestbed.csv -m veos.vtb add-topo vms-kvm-t0 password.txt
```

## Deploy minigraph on the DUT

```
$ ./testbed-cli.sh -t vtestbed.csv -m veos.vtb deploy-mg vms-kvm-t0 lab password.txt
```

You should be login into the sonic kvm using IP: 10.250.0.101 using admin:password.
You should see BGP sessions up in sonic.

```
admin@vlab-01:~$ show ip bgp sum
BGP router identifier 10.1.0.32, local AS number 65100
RIB entries 12807, using 1401 KiB of memory
Peers 8, using 36 KiB of memory
Peer groups 2, using 112 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.0.57       4 64600    3208      12        0    0    0 00:00:22     6400
10.0.0.59       4 64600    3208     593        0    0    0 00:00:22     6400
10.0.0.61       4 64600    3205     950        0    0    0 00:00:21     6400
10.0.0.63       4 64600    3204     950        0    0    0 00:00:21     6400
```
