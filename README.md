# ocp4-upi-install

These playbooks are designed to perform a bare metal install of OCP 4.5 on Intel NUCs. Instructions are tailored for a bare metal install using 4 machines, where one machine hosts the services needed for the cluster and the other three are control plane nodes. 

The ansible playbooks are designed to:
- Generate the necessary configs and install all services on the service node. 
- Download all the necessary RHCOS and OCP install packages.
- Generate the ignition files and set them up to be served from the service node's httpd service, along with the rhcos kernel.

The general networking is as follows:

- network: 192.168.1.0/24, mask: 255.255.255.0, gateway: 192.168.1.1, dns: 192.168.1.11, 8.8.8.8
- domain: snimmo.com
- nuc1.snimmo.com - 192.168.1.11
- nuc2.snimmo.com - 192.168.1.12
- nuc3.snimmo.com - 192.168.1.13
- nuc4.snimmo.com - 192.168.1.14

# Prerequisites

- Install RHEL 8.2 on services machine. (192.168.1.11)
    - Partitioning
        - / - 100 GB
        - /home - 100 GB
        - /nfs - the rest! This will be used as an NFS server for OCP storage
- Copy pub key to use for login
    - `ssh-copy-id -i ~/.ssh/nimmo_rsa.pub root@192.168.1.11`

- Setup a network bridge for libvirt to use for the bootstrap node.
```
nmcli connection add type bridge con-name br0 ifname br0
nmcli connection add type ethernet slave-type bridge con-name br0-eno1 ifname eno1 master br0
nmcli connection modify br0 ipv4.addresses '192.168.1.11/24'
nmcli connection modify br0 ipv4.gateway '192.168.1.1'
nmcli connection modify br0 ipv4.dns '192.168.1.11 8.8.8.8'
nmcli connection modify br0 ipv4.method manual
nmcli connection modify br0 bridge.priority '16384'
nmcli connection up br0 && nmcli connection down eno1
```

# Quick Instructions for Use

- Copy cluster_example.yml from /templates to the root of the project.
- Rename cluster_example.yml to cluster.yml
- Update the cluster.yml with the cluster information for your install.
- Execute `ansible-playbook -e @cluster.yml main.yml --ask-become-pass` from the root of the project.

## Checking the Configuration

- Create Configuration Files for Service node - `ansible-playbook -e @cluster.yml generate_inventory.yml generate_config.yml`
    - This creates a '.config' directory in the root of the project with the generated configs in the folders relative to where they should be put on the services node.

# Start the Bootstrap VM

From the services node, start the virtual machine. You will need to be logged in to the UI of the server. Run the command below, then open the Virtual Machine Manager, select the VM to open. 

`virt-install --name=bootstrap --vcpus=4 --ram=8192 --disk path=/nfs/libvirt/images/bootstrap.qcow2,size=120,bus=virtio --os-variant rhel8.0 --network bridge=br0,model=virtio --boot menu=on --cdrom /ocp/downloads/installer.iso`

Once booted, press tab on the boot menu and append the following to the end of the boot options. 

> These need to be entered on one line, no CR. They are formatted for your readability. 

```
ip=192.168.1.10::192.168.1.1:255.255.255.0:bootstrap.snimmo.com::none
nameserver=192.168.1.11 
coreos.inst.install_dev=vda 
coreos.inst.image_url=http://192.168.1.11:8008/rhcos/metal.raw.gz
coreos.inst.ignition_url=http://192.168.1.11:8008/ignition/bootstrap.ign
```

`/ocp/openshift-install wait-for bootstrap-complete --dir=/ocp/install --log-level=debug`

Once you see this.

`INFO Waiting up to 40m0s for bootstrapping to complete...`

Proceed to install the control plane machines. 

# Install RHCOS on Control Plane Machines

Download the RHCOS installer and flash a USB drive. 

Boot into your instance using the RHCOS ISO Installer

Once booted, press tab on the boot menu and append the following to the end of the boot options. 

> These need to be entered on one line, no CR. They are formatted for your readability. 

> The example below is for the first control plane machine.

```
ip=192.168.1.12::192.168.1.1:255.255.255.0:nuc2.snimmo.com::none
nameserver=192.168.1.11 
coreos.inst.install_dev=sda 
coreos.inst.image_url=http://192.168.1.11:8008/rhcos/metal.raw.gz
coreos.inst.ignition_url=http://192.168.1.11:8008/ignition/master.ign
```

Do that 2 more times for the other two machines, changing the ip and hostname respectively. 

Then wait for the currently executing bootstrap-complete command to say it's successful.

`INFO It is now safe to remove the bootstrap resources`

# Wait for the Install

From the services node.

`/ocp/openshift-install wait-for install-complete --dir=/ocp/install --log-level=debug`

While you are waiting...

`vim /etc/haproxy/haproxy.cfg`

Remove the bootstrap entries. 

Then go delete your bootstrap VM and release the disk space. 