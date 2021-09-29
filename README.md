# Setup of an OKD 4 cluster in a home lab
In this article we will discuss the setup of an OKD4 cluster on a SoC in a home lab.

First I have to point out two other articles which have been used to set up the first cluster at home:
- "OKD 4 Single Node Cluster" by cgruver: https://cgruver.github.io/okd4-single-node-cluster/
- "Guide: Installing an OKD 4.5 Cluster" by Craig Robinson: https://itnext.io/guide-installing-an-okd-4-5-cluster-508a2631cbee

This article combines my findings and several scripts of these two tutorials.

The requirements for this setup are a little knowledge of Linux, how networking works (especially DNS)
and of course the appropriate hardware.

## Architecture

### Hardware

We are going to use a [SoC](https://en.wikipedia.org/wiki/SOC) with 64 Gb RAM and a 1 Tb SSD. This is the
list of the Hardware which is used:
- [Gigabyte GB-BRR7H-4800](https://www.amazon.de/Gigabyte-GB-BRR7H-4800-Barebone-Workstation-schwarz/dp/B095931TWF)
- [Crucial Ballistix 3200 MHz DDR4 DRAM (64 Gb kit)](https://www.amazon.com/dp/B083VV2M8M/ref=cm_sw_em_r_mt_dp_JPS20BDR54MA4F019M56)
- [Western Digital 1TB WD Red SA500 NAS](https://www.amazon.com/dp/B07YFG3R5N/ref=cm_sw_em_r_mt_dp_YWF4X58H9XRCJF2286P1)

Those links are not sponsored or anything. Actually the hardware was purchased by local resellers. But in the
opinion of the author those links are the most recognizable.

### OKD Setup

- `okd4-services` is the host system on the bare metal SoC
- `okd4-bootstrap` the initial node which is used to bootstrap the control-planes
- `okd4-control-plane-1` is running in a virtual machine (KVM)
- `okd4-control-plane-2` is the second virtual machine
- `okd4-control-plane-3` is the third virtual machine

All control planes (aka "master") will get 16 Gb of memory and 120 Gb of hdd space.

In case you are wondering where the worker nodes are: yeah, actually there is not enough memory on the SoC
to set them up. Probably we are going to set them up in an update on a separate machine in the future.

### Network layout

The home network where this is set up already has an own DNS server. We are not going to introduce
new VLANs, we are just going to add the new machines to the existing subnet. Although we will be using
a custom subdomain for all OKD4 related hosts.

We are going to use fixed IP addresses for our setup. So it would be fine to use an IP range which is not
automatically assigned by the home network DHCP server.

Two notes:
- Maybe the security settings on the router regarding MAC address permissions are relevant. Since the
virtual machines will be bridged to the home network it might be necessary to grant them actual 
access to the network.
- The network setup might not work with a WiFi connection - in this tutorial we are using the cable
connection.

## Setup

### Host machine (`okd4-services`)

We are using CentOS 8 as our host system. At the time of writing the version is 8.

Download the iso from http://isoredirect.centos.org/centos/8-stream/isos/x86_64/ and write it to a USB
stick to boot up the system. For this part of the setup a monitor, keyboard and mouse are required. After
the setup is complete we are going to use SSH to connect to the server.

During the installation process the assigned IP address has been used. After completion and before starting
the next steps we will assign a fixed IP address via the DHCP server. If there is no possibility to do that
we can just assign a fixed IP on the host.

### Installation process

Before we start the setup we add an entry to our DHCP server to assign a fixed IP address to the host.
Regarding the selected hostname and domain/subdomain below it will look like this:

```
host okd4-services.my-okd. {
  hardware ethernet xx:xx:xx:xx:xx:xx;
  fixed-address 192.168.10.100;
  option domain-name "my-okd.mylab.home.net";
  ddns-domainname "my-okd.mylab.home.net";
}
```

This should be done for all master nodes as well later on.

We can also prepare the entries on the home lab DNS server to forward all DNS requests to the my-okd
domain to the DNS server on the okd4-services machine:

```
$ORIGIN mylab.home.net.
my-okd              NS      okd4-services.my-okd
$ORIGIN my-okd.mylab.home.net.
okd4-services       A       192.168.10.100
```

#### Hostname

We are going to use `okd4-services.<okd-subdomain>.<home-domain>`.

Which means if our home network domain is `mylab.home.net` and the desired name for the okd subnet `my-okd` 
we are going to use `okd4-services.my-okd.mylab.home.net` as name for the base installation.

#### Partitioning

Choose custom under "Storage Configuration" when selecting the installation destination. After selecting
that and confirming with "Done" we can select the partitioning scheme. Here we choose "Standard partition".

After that we can select "Click here to create them automatically". Here we can delete the home partition
and assign the rest of the space to the root partition. The other automatically generated partitions are
fine.

#### Software selection

We can choose either the "Server with GUI" (if we really need to google something on this machine ;-))
but just "Server" should be fine as well since we are going to manage the system via SSH.

#### Users

Assign a password to the root user. For the remote login we prefer it to add a user to ssh and sudo
the rest of the commands. Just for security reasons. The system will work well with just the root user.

### Configuration and software

Now we have to install the relevant software to set up our okd4 cluster.

```
dnf -y module install virt
dnf -y install wget git net-tools bind bind-utils bash-completion rsync libguestfs-tools virt-install epel-release libvirt-devel httpd-tools nginx
```

Most importantly the virtual machine (KVM), some net tools and nginx to host the files to set up the
okd nodes.

To initialize KVM we have to issue the following commands:

```
systemctl enable libvirtd --now

mkdir /VirtualMachines
virsh pool-destroy default
virsh pool-undefine default
virsh pool-define-as --name default --type dir --target /VirtualMachines
virsh pool-autostart default
virsh pool-start default
```

On a new system the `pool-destroy` and `pool-undefine` commands are not required and probably will result
in an error message.

Next we enable nginx and create the directory where we will host the installation files.

```
systemctl enable nginx --now
mkdir -p /usr/share/nginx/html/install/fcos/ignition
```

Then we need to allow access to these files through the firewall.

```
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
```

Next we create a SSH key pair. This is required to communicate between the OKD nodes and the host later on.
As we can see we are working as root now.

```
ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519
```

#### Network bridge

To set up the virtual machines we have to create a network bridge. Therefore, we have to find the name
of our primary network interface card. This can be achieved by typing `ip addr`. Most likely the card with
an IP address is the primary card (since we are only plugged in via one cable).

In this case this is `enp2s0`.

So define the bridge:

```
nmcli connection add type bridge ifname br0 con-name br0 ipv4.method manual ipv4.address "192.168.10.100/24" ipv4.gateway "192.168.10.1" ipv4.dns "192.168.10.100" ipv4.dns-search "my-okd.mylab.home.net" ipv4.never-default no connection.autoconnect yes bridge.stp no ipv6.method ignore 
```

Mention the DNS entry for the bridge: it is the same as the host. This is because we will be serving
our own DNS server on this machine. And this will be the point where the IP address of the host will
be set to a fixed value!

Now assign the bridge to the primary NIC.

```
nmcli con add type ethernet con-name br0-bind-1 ifname enp2s0 master br0
```

The original NIC binding is now irrelevant, so we delete it.

```
nmcli con del enp2s0
```

Recreate it for the bridge

```
nmcli con add type ethernet con-name enp2s0 ifname enp2s0 connection.autoconnect no ipv4.method disabled ipv6.method ignore
```

In case the IP has changed to the assigned one you will have to log in via SSH with the new IP address.
And as a side note: the system currently can not reach the internet since the DNS resolution works only
for the local machine. We are going to fix that in the next chapter.

#### DNS entries

First we have to edit `/etc/named.conf` and replace the content with the following

```
acl "trusted" {
        192.168.10.0/24;
        127.0.0.1;
};

options {
        listen-on port 53 { 127.0.0.1; 192.168.10.100; };
        
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { trusted; };

        recursion yes;

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.iscdlv.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        forwarders {
                192.168.10.10;
        };
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
include "/etc/named/named.conf.local";
```

This will configure bind to listen on the localhost and the assigned IP address. Mind the `forwarders`
section. This entry allows DNS resolution outside the local machine. In this example we assume that
the network has an own DNS server. If there is no such thing we can safely use `8.8.8.8`.

After that we need to create `/etc/named/named.conf.local` which is referenced above.

```
zone "my-okd.mylab.home.net" {
    type master;
    file "/etc/named/zones/db.my-okd.mylab.home.net"; # zone file path
};

zone "10.168.192.in-addr.arpa" {
    type master;
    file "/etc/named/zones/db.my-okd_ptr";
};
```

Create the PTR entries in `/etc/named/zones/db.my-okd_ptr`. This includes the IP addresses of our
control-plane nodes.

```
@       IN      SOA     okd4-services.my-okd.mylab.home.net. admin.my-okd.mylab.home.net. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; name servers - NS records
      IN      NS      okd4-services.my-okd.mylab.home.net.

; PTR Records
100     IN      PTR     okd4-services.my-okd.mylab.home.net.
110     IN      PTR     okd4-bootstrap.my-okd.mylab.home.net.
101     IN      PTR     okd4-control-plane-1.my-okd.mylab.home.net.
102     IN      PTR     okd4-control-plane-2.my-okd.mylab.home.net.
103     IN      PTR     okd4-control-plane-3.my-okd.mylab.home.net.
```

Finally, we create the DNS entries for all cluster members in `/etc/named/zones/db.my-okd.mylab.home.net`.

```
@	IN	SOA     okd4-services.my-okd.mylab.home.net. admin.my-okd.mylab.home.net. (
             3          ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
;
; name servers - NS records
    IN      NS     okd4-services.my-okd.mylab.home.net.

; name servers - A records
okd4-services.my-okd.mylab.home.net.         IN      A      192.168.10.100

; Cluster A records
okd4-bootstrap.my-okd.mylab.home.net.  IN      A      192.168.10.110
okd4-control-plane-1.my-okd.mylab.home.net.     IN	A      192.168.10.101
okd4-control-plane-2.my-okd.mylab.home.net.     IN	A      192.168.10.102
okd4-control-plane-3.my-okd.mylab.home.net.     IN	A      192.168.10.103

; Internal cluster IPs
etcd-0.okd.my-okd.mylab.home.net.              IN	   A	  192.168.10.101
etcd-1.okd.my-okd.mylab.home.net.              IN	   A	  192.168.10.102
etcd-2.okd.my-okd.mylab.home.net.              IN	   A	  192.168.10.103
*.apps.okd.my-okd.mylab.home.net.     IN	  A	 192.168.10.100
api.okd.my-okd.mylab.home.net.        IN	  A	 192.168.10.100
api-int.okd.my-okd.mylab.home.net.    IN	  A	 192.168.10.100
console-openshift-console.apps.okd.my-okd.mylab.home.net.   IN	A	192.168.10.100
oauth-openshift.apps.okd.my-okd.mylab.home.net.     IN	A	192.168.10.100

_etcd-server-ssl._tcp.okd.my-okd.mylab.home.net.    86400     IN    SRV     0    10    2380    etcd-0.okd
_etcd-server-ssl._tcp.okd.my-okd.mylab.home.net.    86400     IN    SRV     0    10    2380    etcd-1.okd
_etcd-server-ssl._tcp.okd.my-okd.mylab.home.net.    86400     IN    SRV     0    10    2380    etcd-2.okd
```

When this configuration is completed the internet should be reachable again, and we can apply the latest
updates to the system.

```
dnf -y update && init 6
```

#### HA Proxy

We have to set up HA Proxy to route requests between the different nodes.

```
dnf -y install haproxy
```

After installation, we configure HA Proxy

```
sudo setsebool -P haproxy_connect_any 1
sudo systemctl enable haproxy
sudo systemctl start haproxy

sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
```

HA Proxy is required to route the requests to the different nodes on the running cluster.

`/etc/haproxy/haproxy.cfg`

```
# Global settings
#---------------------------------------------------------------------
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          300s
    timeout server          300s
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 20000

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /

frontend okd4_k8s_api_fe
    bind :6443
    default_backend okd4_k8s_api_be
    mode tcp
    option tcplog

backend okd4_k8s_api_be
    balance source
    mode tcp
    server      okd4-bootstrap 192.168.10.110:6443 check
    server      okd4-control-plane-1 192.168.10.101:6443 check
    server      okd4-control-plane-2 192.168.10.102:6443 check
    server      okd4-control-plane-3 192.168.10.103:6443 check

frontend okd4_machine_config_server_fe
    bind :22623
    default_backend okd4_machine_config_server_be
    mode tcp
    option tcplog

backend okd4_machine_config_server_be
    balance source
    mode tcp
    server      okd4-bootstrap 192.168.10.110:22623 check
    server      okd4-control-plane-1 192.168.10.101:22623 check
    server      okd4-control-plane-2 192.168.10.102:22623 check
    server      okd4-control-plane-3 192.168.10.103:22623 check

## for master-only configuration route ingress to master,worker nodes
frontend okd4_http_ingress_traffic_fe
    bind :80
    default_backend okd4_http_ingress_traffic_be
    mode tcp
    option tcplog

backend okd4_http_ingress_traffic_be
    balance source
    mode tcp
    server      okd4-control-plane-1 192.168.10.101:80 check
    server      okd4-control-plane-2 192.168.10.102:80 check
    server      okd4-control-plane-3 192.168.10.103:80 check

frontend okd4_https_ingress_traffic_fe
    bind *:443
    default_backend okd4_https_ingress_traffic_be
    mode tcp
    option tcplog

backend okd4_https_ingress_traffic_be
    balance source
    mode tcp
    server      okd4-control-plane-1 192.168.137.10:443 check
    server      okd4-control-plane-2 192.168.137.10:443 check
    server      okd4-control-plane-3 192.168.137.10:443 check
```

#### Preparations for the installation of the nodes

As a final preparation step we create the directory `install-dir` which serves some files we are going
to need to set up the nodes. The directory is located in the `/root` directory.

We have to put the installation configuration file in this directory: `install-config.yaml`

```
apiVersion: v1
baseDomain: my-okd.mylab.home.net
metadata:
  name: okd
networking:
  networkType: OpenShiftSDN
  clusterNetwork:
  - cidr: 10.100.0.0/14 
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
pullSecret: '{"auths":{"fake":{"auth": "Zm9vOmJhcgo="}}}'
sshKey: %%SSH_KEY%%
```

To replace the ssh key in this file we export and replace it afterwards.

```
SSH_KEY=$(cat ~/.ssh/id_ed25519.pub)

sed -i "s|%%SSH_KEY%%|${SSH_KEY}|g" install-dir/install-config.yaml
```

And create a backup of this file.

`cp install-dir/install-config.yaml install-dir/install-config.yaml.bak`

We also have to create the ignition files for the bootstrap and master nodes.

Here is a script to create MAC addresses:

```
MAC=$(date +%s | md5sum | head -c 6 | sed -e 's/\([0-9A-Fa-f]\{2\}\)/\1:/g' -e 's/\(.*\):$/\1/' | sed -e 's/^/52:54:00:/')
```

It would be great to assign the MAC addresses to a static IP at this point and add a whitelist entry
on your router.

Replace the MAC address in the following scripts.

`install-dir/ignition/bootstrap.yml`:

```
variant: fcos
version: 1.1.0
ignition:
  config:
    merge:
      - local: bootstrap.ign
storage:
  files:
    - path: /etc/zincati/config.d/90-disable-feature.toml
      mode: 0644
      contents:
        inline: |
          [updates]
          enabled = false
    - path: /etc/systemd/network/25-nic0.link
      mode: 0644
      contents:
        inline: |
          [Match]
          MACAddress=ab:cd:ef:11:22:10
          [Link]
          Name=nic0
    - path: /etc/NetworkManager/system-connections/nic0.nmconnection
      mode: 0600
      overwrite: true
      contents:
        inline: |
          [connection]
          type=ethernet
          interface-name=nic0
          [ethernet]
          mac-address=ab:cd:ef:11:22:10
          [ipv4]
          method=manual
          addresses=192.168.10.110/255.255.255.0
          gateway=192.168.10.1
          dns=192.168.10.100
          dns-search=my-okd.mylab.home.net
    - path: /etc/hostname
      mode: 0420
      overwrite: true
      contents:
        inline: |
          okd4-bootstrap.my-okd.mylab.home.net
```

The MAC address is free to choose. It should be one which is not already present in the local network.

`install-dir/ignition/control-plane-1.yml`:

```
variant: fcos
version: 1.1.0
ignition:
  config:
    merge:
      - local: master.ign
storage:
  files:
    - path: /etc/zincati/config.d/90-disable-feature.toml
      mode: 0644
      contents:
        inline: |
          [updates]
          enabled = false
    - path: /etc/systemd/network/25-nic0.link
      mode: 0644
      contents:
        inline: |
          [Match]
          MACAddress=ab:cd:ef:11:22:01
          [Link]
          Name=nic0
    - path: /etc/NetworkManager/system-connections/nic0.nmconnection
      mode: 0600
      overwrite: true
      contents:
        inline: |
          [connection]
          type=ethernet
          interface-name=nic0
          [ethernet]
          mac-address=ab:cd:ef:11:22:01
          [ipv4]
          method=manual
          addresses=192.168.10.101/255.255.255.0
          gateway=192.168.137.1
          dns=192.168.10.100  
          dns-search=my-okd.mylab.home.net
    - path: /etc/hostname
      mode: 0420
      overwrite: true
      contents:
        inline: |
          okd4-control-plane-1.my-okd.mylab.home.net
```

This file needs to be created three times - `control-plane-1`, `control-plane-2` and `control-plane-3`
since we are going to set up three master nodes. Be aware to assign unique MAC addresses.

Download the oc tools. We just need them temporarily since the installation process will download the
actual version and replaces it. The OKD release version could be any version of the 4 stream.

```
wget https://github.com/openshift/okd/releases/download/4.7.0-0.okd-2021-03-07-090821/openshift-client-linux-4.7.0-0.okd-2021-03-07-090821.tar.gz
```

```
tar -xzf openshift-client-linux-4.7.0-0.okd-2021-03-07-090821.tar.gz
mv oc ~/bin
mv kubectl ~/bin
rm -f openshift-client-linux-4.7.0-0.okd-2021-03-07-090821.tar.gz
rm -f README.md
```

```
mkdir -p install-dir/okd-release-tmp
cd isntall-dir/okd-release-tmp
oc adm release extract --command='openshift-install' quay.io/openshift/okd:4.7.0-0.okd-2021-03-07-090821
oc adm release extract --command='oc' quay.io/openshift/okd:4.7.0-0.okd-2021-03-07-090821
mv -f openshift-install ~/bin
mv -f oc ~/bin
cd ..
rm -rf okd-release-tmp
```

Create `install-dir/work-dir/ignition` and copy the ignition files into it.

Download fcct to `install-dir/work-dir`

```
wget https://github.com/coreos/fcct/releases/download/v0.6.0/fcct-x86_64-unknown-linux-gnu
mv fcct-x86_64-unknown-linux-gnu install-dir/work-dir/fcct 
chmod 750 install-dir/work-dir/fcct
```

Create the ignition files with `openshift-install`

```
openshift-install --dir=/root/install-dir create ignition-configs
```

Copy all files to the nginx hosted directory and make them accessible.

```
cp -r /root/install-dir/*.ign /usr/share/nginx/html/install/fcos/ignition
chmod 644 /usr/share/nginx/html/install/fcos/ignition/*
```

Download syslinux

```
cd
curl -o install-dir/syslinux-6.03.tar.xz https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/syslinux-6.03.tar.xz
tar -xf install-dir/syslinux-6.03.tar.xz -C install-dir/
```

Prepare the install images

```
mkdir -p install-dir/fcos-iso/{isolinux,images}
curl -o install-dir/fcos-iso/images/vmlinuz https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210104.3.0/x86_64/fedora-coreos-33.20210104.3.0-live-kernel-x86_64
curl -o install-dir/fcos-iso/images/initramfs.img https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210104.3.0/x86_64/fedora-coreos-33.20210104.3.0-live-initramfs.x86_64.img
curl -o install-dir/fcos-iso/images/rootfs.img https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210104.3.0/x86_64/fedora-coreos-33.20210104.3.0-live-rootfs.x86_64.img
```

Copy the files together

```
cp install-dir/syslinux-6.03/bios/com32/elflink/ldlinux/ldlinux.c32 install-dir/fcos-iso/isolinux/ldlinux.c32
cp install-dir/syslinux-6.03/bios/core/isolinux.bin install-dir/fcos-iso/isolinux/isolinux.bin
cp install-dir/syslinux-6.03/bios/com32/menu/vesamenu.c32 install-dir/fcos-iso/isolinux/vesamenu.c32
cp install-dir/syslinux-6.03/bios/com32/lib/libcom32.c32 install-dir/fcos-iso/isolinux/libcom32.c32
cp install-dir/syslinux-6.03/bios/com32/libutil/libutil.c32 install-dir/fcos-iso/isolinux/libutil.c32
```

### Bootstrap node (`okd4-bootstrap`)

Create the ISO image for the bootstrap node

`install-dir/fcos-iso/isolinux/bootstrap.cfg`

```
serial 0
default vesamenu.c32
timeout 1
menu clear
menu separator
label linux
  menu label ^Fedora CoreOS (Live)
  menu default
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img,/images/rootfs.img net.ifnames=1 ifname=nic0:ab:cd:ef:11:22:10 ip=192.168.10.110::192.168.10.1:255.255.255.0:okd4-bootstrap.my-okd.mylab.home.net:nic0:none nameserver=192.168.10.100 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.10.100/install/fcos/ignition/bootstrap.ign coreos.inst.platform_id=qemu console=ttyS0
menu separator
menu end
```

```
mkisofs -o /tmp/bootstrap.iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -J -r install-dir/fcos-iso/
```

And finally create the VM

```
mkdir -p /VirtualMachines/okd4-bootstrap
virt-install --name okd4-bootstrap --memory 14336 --vcpus 2 --disk size=100,path=/VirtualMachines/okd4-bootstrap/rootvol,bus=sata --cdrom /tmp/bootstrap.iso --network bridge=br0 --mac=ab:cd:ef:11:22:10 --graphics none --noautoconsole --os-variant centos7.0
```

After that the bootstrap node starts up, downloads the image and begins installation. When the installation
is completed the VM shuts down and we can go on configuring the control plane(s).

### Control plames (`okd4-control-plane-[1-3]`)

Create the ISO image here as well by reusing the `install-dir/fcos-iso/isolinux/isolinux.cfg` configuration

Replace the MAC address here as well.

```
serial 0
default vesamenu.c32
timeout 1
menu clear
menu separator
label linux
  menu label ^Fedora CoreOS (Live)
  menu default
  kernel /images/vmlinuz
  append initrd=/images/initramfs.img,/images/rootfs.img net.ifnames=1 ifname=nic0:52:54:00:71:28:7c ip=192.168.10.101::192.168.10.1:255.255.255.0:okd4-control-plane-1.my-okd.mylab.home.net:nic0:none nameserver=192.168.10.100 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.10.100/install/fcos/ignition/master.ign coreos.inst.platform_id=qemu console=ttyS0
menu separator
menu end
```

```
mkisofs -o /tmp/control-plane-1.iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -J -r install-dir/fcos-iso/
```

And create the VM

```
mkdir -p /VirtualMachines/okd4-control-plane-1
virt-install --name okd4-control-plane-1 --memory 16384 --vcpus 4 --disk size=200,path=/VirtualMachines/okd4-control-plane-1/rootvol,bus=sata --cdrom /tmp/control-plane-1.iso --network bridge=br0 --mac=52:54:00:71:28:7c --graphics none --noautoconsole --os-variant centos7.0
```

We have to repeat this for all our three control plane nodes.

### Starting up the system

After the VMs have been set up we can start up the nodes.

```
virsh start okd4-bootstrap
virsh start okd4-control-plane-1
virsh start okd4-control-plane-2
virsh start okd4-control-plane-3
```

To watch the installation progress we can run the following command

```
openshift-install --dir=/root/install-dir wait-for bootstrap-complete --log-level debug
```

Meanwhile, one could log in to the bootstrap node to see what's going on

```
ssh core@okd4-bootstrap.my-okd.mylab.home.net
```

When the `openshift-install` command logs that the bootstrap process is completed like that (some errors
can occur)

```
DEBUG OpenShift Installer 4.7.0-0.okd-2021-03-07-090821 
DEBUG Built from commit a005bb9eddcbc97e4cac2cdf4436fe2d524cc75e 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.okd.fluff.chef.at:6443... 
DEBUG Still waiting for the Kubernetes API: an error on the server ("") has prevented the request from succeeding 
DEBUG Still waiting for the Kubernetes API: an error on the server ("") has prevented the request from succeeding 
DEBUG Still waiting for the Kubernetes API: an error on the server ("") has prevented the request from succeeding 
INFO API v1.20.0-1046+5fbfd197c16d3c-dirty up     
INFO Waiting up to 30m0s for bootstrapping to complete... 
DEBUG Bootstrap status: complete                   
INFO It is now safe to remove the bootstrap resources 
DEBUG Time elapsed per stage:                      
DEBUG Bootstrap Complete: 26m42s                   
DEBUG                API: 6m15s                    
INFO Time elapsed: 26m42s
```

The bootstrap node can be disabled in `haproxy` by commenting out the bootstrap nodes.

```
sudo sed '/ okd4-bootstrap /s/^/#/' /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

Disable nginx since we will be using haproxy to route the traffic to the nodes. In this master-only-setup
we are going to route all the http and https ingress traffic to the control-planes.

If we need the hosted ignition files again later on we should assign port 8080 to nginx and add the firewall
rule.

haproxy config as initially mentioned above:

```
...
backend okd4_k8s_api_be
    balance source
    mode tcp
#    server      okd4-bootstrap 192.168.10.110:6443 check
    server      okd4-control-plane-1 192.10.137.101:6443 check
    server      okd4-control-plane-2 192.10.137.102:6443 check
    server      okd4-control-plane-3 192.10.137.103:6443 check
...

...
backend okd4_machine_config_server_be
    balance source
    mode tcp
#    server      okd4-bootstrap 192.168.10.110:22623 check
    server      okd4-control-plane-1 192.168.10.101:22623 check
    server      okd4-control-plane-2 192.168.10.102:22623 check
    server      okd4-control-plane-3 192.168.10.103:22623 check
...
```

Disable nginx and reload haproxy:

```
systemctl stop nginx.service
systemctl disable nginx.service
systemctl reload haproxy.service
```

As for now it should be possible to log in to the cluster via
`https://console-openshift-console.apps.okd.my-okd.mylab.home.net/` using the username `kubeadmin` with
the password provided in `/root/install-dir/auth/kubeadmin-password`.

### Login via oc command line

To perform actions via the `oc` command the `KUBEADMIN` variable needs to be exported:

```
export KUBECONFIG="/root/install-dir/auth/kubeconfig"
```

When using this you are logged in as kubeadmin.

## Next steps

### Creating Persistent Volume Claims (PVC)

First we have to set up the nfs service on our okd4-services machine. Probably `nfs-utils` is already
installed.

```
dnf install -y nfs-utils
systemctl enable nfs-server rpcbind
systemctl start nfs-server rpcbind
mkdir -p /var/nfsshare/registry
chmod -R 777 /var/nfsshare
chown -R nobody:nobody /var/nfsshare
```

Export the created directory via nfs share

```
echo '/var/nfsshare 192.168.10.0/24(rw,sync,no_root_squash,no_all_squash,no_wdelay)' | sudo tee /etc/exports
```

Then the service needs to be restarted and firewall rules have to be added:

```
sudo setsebool -P nfs_export_all_rw 1
sudo systemctl restart nfs-server
sudo firewall-cmd --permanent --zone=public --add-service mountd
sudo firewall-cmd --permanent --zone=public --add-service rpc-bind
sudo firewall-cmd --permanent --zone=public --add-service nfs
sudo firewall-cmd --reload
```

When this is completed we have to add the persistent volume to the OKD configuration. Therefore, a file
`/root/install-dir/work-dir/registry_pv.yaml` is needed.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /var/nfsshare/registry
    server: 192.168.10.100
```

Apply it with

```
oc create -f /root/install-dir/work-dir/registry_pv.yaml
```

When listing the persistent volumes with `oc get pv` it shows something like this:

```
> oc get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
registry-pv   100Gi      RWX            Retain           Available                                   4s
```

Now we have to modify the image-registry operator:

```
oc edit configs.imageregistry.operator.openshift.io
```

Edit the `spec` section and change the value of `managementState` from `Never` to `Managed` and add empty
entries to the `storage` section here:

```
spec:
# ...
  managementState: Managed
# ...
  storage:
    pvc:
      claim:
# ...
```

After saving the configuration we can run `oc get pv` again and should see this result:

```
> oc get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS   REASON   AGE
registry-pv   100Gi      RWX            Retain           Bound    openshift-image-registry/image-registry-storage                           4m28s
```

### Adding custom users

Here we are using a htpasswd "identity provider". We are going to just create a htpasswd file and add users
to it for simplicity. Of course, it is possible to use other identity providers.

```
htpasswd -cB /root/install-dir/work-dir/htpasswd admin
htpasswd -b /root/install-dir/work-dir/htpasswd devuser
```

We then create the secret in OKD via the command line:

```
oc create -n openshift-config secret generic htpasswd-secret --from-file=htpasswd=/root/install-dir/work-dir/htpasswd
```

Then we apply the according configuration to use this secret as identity provider. Therefore, we create
the yaml file `/root/install-dir/work-dir/htpasswd-cr.yaml`:

```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: okd4_htpasswd_idp
    mappingMethod: claim 
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret
```

```
oc apply -f /root/install-dir/work-dir/htpasswd-cr.yaml
```

Finally, we are going to make the admin user a cluster admin:

```
oc adm policy add-cluster-role-to-user cluster-admin admin
```

When this is done we can log in to the console choosing the okd4_htpasswd_idp and using the credentials
of one of the two created users.

### Providing custom certificates

We have created an `host.ext` file to create x509v3 certificates which is referenced in the scripts below.

```
extensions = x509v3

 [ x509v3 ]
nsCertType              = server
keyUsage                = digitalSignature,nonRepudiation,keyEncipherment
extendedKeyUsage        = serverAuth
```

Default settings are stored in `host.conf`:

```
 [ req ]
default_bits               = 4096
distinguished_name         = req_DN
string_mask                = nombstr

 [req_DN ]
countryName                = "1. Country Name      (2 letter code)"
countryName_default        = AT
countryName_min            = 2
countryName_max            = 2
stateOrProvinceName        = "2. State or Province Name (full Name)"
stateOrProvinceName_default = Vienna
localityName               = "3. Locality Name (eg. City)"
localityName_default       = Vienna
0.organizationName         = "4. Organization Name (eg. Company)"
0.organizationName_default = My Lab
organizationalUnitName     = "5. Organizational Unit Name (eg. section)"
organizationalUnitName_default  = OKD services
commonName                 = "6. Common Name (eg. CA Name)"
commonName_max             = 64
commonName_default         = HOST.net
emailAddress               = "7. Email Address (eg. name@FQDN)"
emailAddress_max           = 40
emailAddress_default       = admin@HOST.net
```

Later on it is important for the certificate to set the common name to the domain.

When creating the certificates later on we can see that we are using a custom CA and an intermediate CA
to sign the certificate. We can create those certificates in advance. This has the advantage that we can
import our own CA in the OS truststore and trust all our self-signed certificates.

#### Creating the certificate for the api

As the [OKD docs](https://docs.okd.io/latest/security/certificates/api-server.html) mention, never ever
create a certificate for the api-int domain!

This certificate can be created directly for the domain. No wildcard alias is needed. To allow OKD to
read the key we assign no password.

```
openssl req \
        -new \
        -nodes \
        -reqexts SAN \
        -extensions SAN \
        -config <(cat host.conf <(printf "\n[SAN]\nsubjectAltName=DNS:api.okd.my-okd.mylab.net\n")) \
        -out api.okd.my-okd.mylab.net.csr \
        -keyout api.okd.my-okd.mylab.net.key \
        -newkey rsa:4096 \
        -sha512
```

This creates the private key and a certificate request. We are going to sign the certificate request:

```
openssl x509 -req \
        -extfile <(cat host.ext <(printf "\nsubjectAltName=DNS:api.okd.my-okd.mylab.net\n")) \
        -in api.okd.my-okd.mylab.net.csr \
        -CAkey intermediate/ca-int.key \
        -CA intermediate/ca-int.crt \
        -days 825 \
        -sha512 \
        -set_serial $RANDOM \
        -out api.okd.my-okd.mylab.net.crt
```

And finally we create one file with the certificate chain containing our root CA, intermediate CA and the
certificate for the domain itself.

```
cat api.okd.my-okd.mylab.net.crt intermediate/ca-int.crt ca/ca.crt > api.okd.my-okd.mylab.net.chained.crt
```

With those certificates we create a secret in our OKD cluster:

```
oc create secret tls ssl-api-cert \
     --cert=api.okd.my-okd.mylab.net.chained.crt \
     --key=api.okd.my-okd.mylab.net.key \
     -n openshift-config
```

Then we have to patch the configuration of our api server:

```
oc patch apiserver cluster \
     --type=merge -p \
     '{"spec":{"servingCerts": {"namedCertificates":
     [{"names": ["api.okd.my-okd.mylab.net"], 
     "servingCertificate": {"name": "ssl-api-cert"}}]}}}'
```

After a short time the certificate should be applied. This can be tested by calling
`https://api.okd.my-okd.mylab.net:6443` in a browser. Or when the CA is imported in the OS truststore
we can call `oc whoami` from our workstation (when logged in) and see if there is a certificate error.

When the certificate is not recognized one have to log in with a login command from the okd console
(placed under the user menu -> copy login command) and allow the unrecognized certificate with oc login.

#### Creating the certificate for the apps domain

The creation of this certificate is pretty much the same as for the api. But in this case we assign a
wildcard domain as DNS alternative name to allow all subdomains under `apps.okd.my-okd.mylab.net` to use
the same certificate. We do this here for simplicity.

Mind the asterisk in the following commands when assigning the DNS alternative name.

Request:

```
openssl req \
        -new \
        -nodes \
        -reqexts SAN \
        -extensions SAN \
        -config <(cat host.conf <(printf "\n[SAN]\nsubjectAltName=DNS:*.apps.okd.my-okd.mylab.net\n")) \
        -out apps.okd.my-okd.mylab.net.csr \
        -keyout apps.okd.my-okd.mylab.net.key \
        -newkey rsa:4096 \
        -sha512
```

Sign:

```
openssl x509 -req \
        -extfile <(cat host.ext <(printf "\nsubjectAltName=DNS:*.apps.okd.my-okd.mylab.net\n")) \
        -in apps.okd.my-okd.mylab.net.csr \
        -CAkey intermediate/ca-int.key \
        -CA intermediate/ca-int.crt \
        -days 825 \
        -sha512 \
        -set_serial $RANDOM \
        -out apps.okd.my-okd.mylab.net.crt
```

Chain:

```
cat api.okd.my-okd.mylab.net.crt intermediate/ca-int.crt ca/ca.crt > api.okd.my-okd.mylab.net.chained.crt
```

The description to apply the certificate for our ingress routes comes from
[https://docs.okd.io/latest/security/certificates/replacing-default-ingress-certificate.html](https://docs.okd.io/latest/security/certificates/replacing-default-ingress-certificate.html).

First we add the root CA used to sign our certificates as an config map entry:

```
oc create configmap custom-ca \
     --from-file=ca-bundle.crt=ca/ca.crt \
     -n openshift-config
```

Then patch the proxy/cluster configuration:

```
oc patch proxy/cluster \
     --type=merge \
     --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'
```

Create the secret for the ingress namespace:

```
oc create secret tls ssl-ingress-cert \
     --cert=apps.okd.my-okd.mylab.net.cained.crt \
     --key=apps.okd.my-okd.mylab.net.key \
     -n openshift-ingress
```

And finally patch the ingress controller:

```
oc patch ingresscontroller.operator default \
     --type=merge -p \
     '{"spec":{"defaultCertificate": {"name": "ssl-ingress-cert"}}}' \
     -n openshift-ingress-operator
```

After this command has been issued it could be possible that the cluster is not reachable for some time.
But when all operators and the control planes are back up and synced the cluster will be fine again.
