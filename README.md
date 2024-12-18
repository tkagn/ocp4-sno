# ocp4-sno
Single node OpenShift Quick deployment details

# SNO Deployment

## Requirements

- At least 8 vCPUs
- 32 GB of RAM
- 120 GB of disk space

## Bastion

bastion.lab.tkagn.io

```bash
virt-install -v -n bastion.lab.tkagn.io --vcpus 4 --memory 4096 --os-variant fedora39 --import --disk 'path=/var/lib/libvirt/images/bastion.qcow2' --disk 'device=cdrom,path=/var/lib/libvirt/images/bastion-cloudconfig.iso,bus=scsi' --network network=default,model=virtio --network bridge=br0-eno1.101,model=virtio --graphics 'none' --console pty,target_type=serial --noautoconsole --noreboot
```


192.168.122.48 bastion.lab.tkagn.io bastion

```bash
nmcli connection delete Wired\ connection\ 1
nmcli con add type vlan con-name eth1-vlan100 dev eth1 id 100 ip4 192.168.200.10/24 ipv4.dns 192.168.200.121 ipv4.method manual
nmcli con up eth1-vlan100

cat << EOF >> /etc/systemd/resolved.conf
[Resolve]
DNS=192.168.200.121

EOF
```


## OpenShift Tools

```bash
# `openshift-install`
curl "https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz" -O \
&& tar -xzvf openshift-install-linux.tar.gz -C /usr/local/bin/ \
&& openshift-install completion bash >> /etc/bash_completion.d/openshift-install_bash_completion \
&& rm -f /usr/local/bin/README.md openshift-install-linux.tar.gz
```

## SNO1

 
192.168.200.41 sno1.lab.tkagn.io sno1


## Generate SNO Install Media

### Download RHCoreOS

```bash
curl -L 'https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/latest/rhcos-live.x86_64.iso' -O
mv rhcos-4.16.0-x86_64-live.x86_64.iso snoboot.iso
```

### Download `coreos-installer`

```bash
curl -L 'https://mirror.openshift.com/pub/openshift-v4/clients/coreos-installer/latest/coreos-installer_amd64' -O
cp coreos-installer_amd64 coreos-installer
chmod +x ./coreos-installer
cp ./coreos-installer /usr/local/bin/
```

### Create `install-config.yaml`

```bash
cat << EOF >> ./install-config.yaml
apiVersion: v1
baseDomain: lab.tkagn.io
metadata:
 name: sno1
networking:
 networkType: OVNKubernetes
 machineCIDR: 192.168.200.0/24
 clusterNetwork:
 - cidr: 10.128.0.0/14
   hostPrefix: 23 
 serviceNetwork:
 - 172.30.0.0/16
compute:
- name: worker
  replicas: 0
controlPlane:
 name: master
 replicas: 1
platform:
 none: {}
BootstrapInPlace:
 InstallationDisk: /dev/vda
pullSecret: "PASTE THE SECRET CONTENT HERE"
sshKey: |
 "PASTE THE SSH PUBLIC KEY HERE"
EOF
```

### Generate Ignition File for SNO

```bash
cp ./install-config.yaml ./deploy
openshift-install create single-node-ignition-config --dir=./deploy
```

### Embed Ignition File Into Boot Media

```bash
cp rhcos-4.16.0-x86_64-live.x86_64.iso sno1boot.iso
coreos-installer iso ignition embed -i ./deploy/bootstrap-in-place-for-live-iso.ign sno1boot.iso
```

### Set Static IP Configuration

```bash
coreos-installer iso kargs modify -a "ip=192.168.122.41::192.168.122.1:255.255.255.0:sno1.lab.tkagn.io:ens1:off:192.168.130.1:8.8.8.8" /var/lib/libvirt/images/rhcos-sno-4.8.2-x86_64-live.x86_64.iso
```


## Create SNO VM

```bash
virt-install -v -n sno1.lab.tkagn.io --vcpus 8 --memory 32768 --os-variant rhel9.2 --import --disk 'path=/dev/vmdisks/sno1-os' --cdrom 'snoboot.iso' --network bridge=br0-eno1.101,model=virtio --graphics 'vnc' --console pty,target_type=serial --noautoconsole --noreboot --boot menu=on

virsh autostart sno1.lab.tkagn.io 
```

### Monitor Installation

```bash
openshift-install wait-for install-complete --dir deploy
```

## Reference

- [Installing on a single node](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/installing_on_a_single_node/index)
- [Installing single-node OpenShift manually](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/installing_on_a_single_node/install-sno-installing-sno#install-sno-installing-sno-manually)
- [Openshift running as single node with libvirt kvm](https://fajlinuxblog.medium.com/openshift-running-as-single-node-with-libvirt-kvm-cb615d2c43e6)
