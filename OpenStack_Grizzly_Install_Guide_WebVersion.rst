==========================================================
  OpenStack Grizzly Install Guide
==========================================================

:Version: 1.0
:Source: https://github.com/mseknibilel/OpenStack-Folsom-Install-guide

Table of Contents
=================

::

  1. Requirements
  2. Getting Ready
  3. Keystone 
  4. Glance
  5. Nova
  6. Cinder
  7. Adding a compute node
  8. Start your first VM

1. Requirements
====================

:Node Role: NICs
:Control Node: eth0 (10.111.82.1)
:Compute Node: em1 (10.111.82.2)

**Note:** You can add as many compute node as you wish.

2. Getting Ready
===============

2.1. Preparing Ubuntu 12.04
-----------------

* After you install Ubuntu 12.04 Server 64bits, go to the sudo mode and don't leave it until the end of this guide::

   sudo su

* Add Grizzly repositories [Only for Ubuntu 12.04]::

     apt-get install -y ubuntu-cloud-keyring 
     echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main >> /etc/apt/sources.list.d/grizzly.list

* Update your system::

   apt-get update
   apt-get upgrade -y
   apt-get dist-upgrade -y

2.2.Networking
------------
* First, take a good look at your working routing table::
   
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   0.0.0.0         10.111.82.254   0.0.0.0         UG    0      0        0 eth0
   10.111.82.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
 
* The file /etc/network/interfaces should look like this::

   auto lo
   iface lo inet loopback
 
   auto eth0
   iface eth0 inet static
   address 10.111.82.1
   netmask 255.255.255.0
   network 10.111.82.0
   broadcast 10.111.82.255
   gateway 10.111.82.254
   dns-nameservers 10.1.1.68 10.1.1.42
   dns-search despexds.net

2.3. MySQL & RabbitMQ
------------

* Install MySQL::

   apt-get install mysql-server python-mysqldb

* Configure mysql to accept all incoming requests::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* Permit the user root to connect from everywhere and delete the anonymous user::

   mysql -u root -p
   update mysql.user set host = '%' where host = '::1';
   delete from mysql.user where user = '';
   flush privileges;
   quit;

* Install RabbitMQ::

   apt-get install rabbitmq-server 

2.4. Node synchronization
------------------

* Install other services::

   apt-get install ntp

* Configure the NTP server to synchronize between your compute nodes and the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
   service ntp restart  

2.5. Others
-------------------
* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf 

* Add 8021q to /etc/modules::

   echo "8021q" >> /etc/modules


3. Keystone
=====================================================================

This is how we install OpenStack's identity service:

* Start by the keystone packages::

   apt-get install keystone

* Create a new MySQL database for keystone::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
   quit;

* Adapt the connection attribute in the /etc/keystone/keystone.conf to the new database::

   connection = mysql://keystoneUser:keystonePass@localhost/keystone

* Restart the identity service then synchronize the database::

   service keystone restart
   keystone-manage db_sync

* Fill up the keystone database using the two scripts available in the `Scripts folder <https://github.com/mseknibilel/OpenStack-Grizzly-Install-guide/tree/master/Keystone_Scripts>`_ of this git repository.::

   #Modify the HOST_IP variable before executing the scripts

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* Load the credential data in the file /etc/profile::

   echo '
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://10.111.80.201:5000/v2.0/"
   export OS_NO_CACHE=1' >> /etc/profile
   source /etc/profile

* To test Keystone, we use a simple curl request::

   curl http://10.111.80.201:35357/v2.0/endpoints -H 'x-auth-token: ADMIN'

* Reboot, test connectivity and check Keystone again.

4. Glance
=====================================================================

* After installing Keystone, we continue with installing image storage service (a.k.a Glance)::

   apt-get install glance

* Create a new MySQL database for Glance::

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass!';
   quit;

* Update /etc/glance/glance-api-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.111.82.1
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update the /etc/glance/glance-registry-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.111.82.1
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update /etc/glance/glance-api.conf with::

   sql_connection = mysql://glanceUser:glancePass!@localhost/glance

* And::

   [paste_deploy]
   flavor = keystone

* Update the /etc/glance/glance-registry.conf with::

   sql_connection = mysql://glanceUser:glancePass!@localhost/glance

* And::

   [paste_deploy]
   flavor = keystone

* Restart the glance-api and glance-registry services::

   service glance-api restart; service glance-registry restart

* Synchronize the glance database::

   glance-manage db_sync

* To test Glance, we upload a new image to the store. Start by downloading the cirros cloud image to your node and then uploading it to Glance::

   mkdir images
   cd images
   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
   glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 < cirros-0.3.0-x86_64-disk.img

* Now list the images to see what you have just uploaded::

   glance image-list

* Run the following script, called migrate-to-folsom.sh, to import Despegar's Ubuntu base image::


* Enable the endpoint v1 for Glance in the Keystone database::

   Simply replace "v2" with "v1" in the 'extra' column of the 'endpoint' table in 'keystone' database.
   The row to modify is the one with "id" equal to the "service_id" with 'image' type in the 'service' table.
   In our case is the one whose url shows port 9292.

* Install and configure nfs::

   apt-get -y install nfs-kernel-server
   echo '/var/lib/glance/images 10.0.0.0/8(rw,no_root_squash,subtree_check)' >> /etc/exports
   exportfs -a
   service nfs-kernel-server restart

5. Nova
=================

* Install these packages::

   apt-get install nova-api nova-cert nova-doc nova-scheduler nova-consoleauth

* Prepare a Mysql database for Nova::

   mysql -u root -p
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   quit;

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.111.82.1
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova


* Change your /etc/nova/nova.conf to look like this::

   [DEFAULT]
   
   # LOGS/STATE
   verbose=True
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   
   # AUTHENTICATION
   auth_strategy=keystone
   
   # SCHEDULER
   scheduler_driver=nova.scheduler.multi.MultiScheduler
   compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
   
   # CINDER
   volume_api_class=nova.volume.cinder.API
   
   # DATABASE
   sql_connection=mysql://novaUser:novaPass@10.111.82.1/nova
   
   # COMPUTE
   libvirt_type=kvm
   libvirt_use_virtio_for_bridges=True
   start_guests_on_host_boot=True
   resume_guests_state_on_host_boot=True
   api_paste_config=/etc/nova/api-paste.ini
   allow_admin_api=True
   use_deprecated_auth=False
   nova_url=http://10.111.82.1:8774/v1.1/
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
   
   # APIS
   ec2_host=10.111.82.1
   ec2_url=http://10.111.82.1:8773/services/Cloud
   keystone_ec2_url=http://10.111.82.1:5000/v2.0/ec2tokens
   s3_host=10.111.82.1
   cc_host=10.111.82.1
   metadata_host=10.111.82.1
   #metadata_listen=0.0.0.0
   enabled_apis=ec2,osapi_compute,metadata
   
   # RABBITMQ
   rabbit_host=10.111.82.1
   
   # GLANCE
   image_service=nova.image.glance.GlanceImageService
   glance_api_servers=10.111.82.1:9292
   
   # NETWORK
   network_manager=nova.network.manager.FlatDHCPManager
   force_dhcp_release=True
   dhcpbridge_flagfile=/etc/nova/nova.conf
   dhcpbridge=/usr/bin/nova-dhcpbridge
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   public_interface=eth0
   flat_interface=eth0
   flat_network_bridge=br100
   fixed_range=192.168.6.0/24
   network_size=256
   flat_network_dhcp_start=192.168.6.0
   flat_injected=False
   connection_type=libvirt
   multi_host=True

* Don't forget to update the ownership rights of the nova directory::

   chown -R nova. /etc/nova
   chmod 644 /etc/nova/nova.conf

* Add this line to the sudoers file::

   sudo visudo
   #Paste this line anywhere you like:
   nova ALL=(ALL) NOPASSWD:ALL

* Synchronize your database::

   nova-manage db sync

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

* Use the following command to create fixed network::
   
   nova-manage network create private --fixed_range_v4=192.168.6.0/24 --num_networks=1 --bridge=br100 --bridge_interface=eth0 --network_size=256 --multi_host=T

* Create the floating IPs ranges for both vlans::

   nova-manage floating create --ip_range=10.111.82.128/26 --pool vlan82
   nova-manage floating create --ip_range=10.222.92.128/26 --pool vlan92
   
* Create the floating to the nova project, run the next command many times as your network IPs::

    nova floating-ip-create

* Add ICMP ping and all TCP and UDP access to the default security group::

    nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
    nova secgroup-add-rule default tcp 1 65535 0.0.0.0/0
    nova secgroup-add-rule default udp 1 65535 0.0.0.0/0

6. Cinder
=================

Although Cinder is a replacement of the old nova-volume service, its installation is now a seperated from the nova install process.

* Install the required packages::

   apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* Configure the iscsi services::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* Restart the services::
   
   service iscsitarget start
   service open-iscsi start

* Prepare a Mysql database for Cinder::

   mysql -u root -p
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass!';
   quit;

* Configure /etc/cinder/api-paste.ini like the following::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 10.111.82.1
   service_port = 5000
   auth_host = 10.111.82.1
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* Edit the /etc/cinder/cinder.conf to::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@localhost/cinder
   api_paste_confg = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* Then, synchronize your database::

   cinder-manage db sync

* Finally, don't forget to create a volumegroup and name it cinder-volumes::

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2
   #Type in the followings:
   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* Proceed to create the physical volume then the volume group::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

**Note:** Beware that this volume group gets lost after a system reboot. (Click `Here <https://github.com/mseknibilel/OpenStack-Grizzly-Install-guide/blob/master/Tricks%26Ideas/load_volume_group_after_system_reboot.rst>`_ to know how to load it after a reboot) 

* Restart the cinder services::

   service cinder-volume restart
   service cinder-api restart

7. Miscelaneos
=========================

* Mail settings::

   apt-get install mutt -y

* Edit the archive /etc/postfix/main.cf::

   relayhost = mail01.despexds.net

* Ensure every service of openstack to start after reboot (nova*, glance*, keystone, mysql, cinder*)::

   apt-get install sysv-rc-conf
   sysv-rc-conf

* Estandarizar flavors ejecutando los siguientes comandos::

   nova flavor-delete 1
   nova flavor-delete 2
   nova flavor-delete 3
   nova flavor-delete 4
   nova flavor-delete 5
   nova flavor-create m1.tiny 1 512 0 1
   nova flavor-create ddlg 100 8192 3 4
   nova flavor-create chori 101 2048 3 2
   nova flavor-create disquito 103 4096 7 4
   nova flavor-create medio 104 4096 3 2
   nova flavor-create large.1 105 16384 32 8
   nova flavor-create medium.2 106 8192 20 4
   nova flavor-create tiny.1 107 2048 4 1
   nova flavor-create tiny.2 108 4096 12 1
   nova flavor-create small.1 109 4096 8 2
   nova flavor-create small.2 110 8192 24 2
   nova flavor-create medium.1 111 8192 16 4
   nova flavor-create large.2 112 20480 40 8
   nova flavor-create huge.1 113 32768 80 16
   nova flavor-create huge.2 114 16384 40 16
   nova flavor-create small.3 115 8192 50 2
   nova flavor-create small.2.pd 116 8192 10 2
   nova flavor-create large.1.sd 117 16384 4 8
   nova flavor-create large.3 118 8192 40 8
   nova flavor-create huge.2.sd 119 16384 16 16
   nova flavor-create std.small 121 3072 3 2
   nova flavor-create std.medium 122 8192 8 4
   nova flavor-create std.large 123 16384 16 8
   nova flavor-create std.huge 124 32768 32 16
   nova flavor-create disk.small 125 3072 30 2
   nova flavor-create disk.medium 126 8192 80 4
   nova flavor-create disk.large 127 16384 160 8
   nova flavor-create cpu.medium 128 4096 4 8
   nova flavor-create cpu.large 129 8192 8 16
   nova flavor-create std.tiny 130 1024 3 1
   nova flavor-create mem.small 131 4096 4 2
   nova flavor-create mem.medium 132 12288 12 4
   nova flavor-create mem.large 133 24576 24 8
   nova flavor-create disk.medium2 134 8192 30 4
   nova flavor-create mem.huge 135 65536 64 16
   nova flavor-create std2.tiny 136 2048 7 1
   nova flavor-create std2.small 137 4096 9 2
   nova flavor-create std2.medium 138 8192 13 4
   nova flavor-create std2.large 139 16384 21 8
   nova flavor-create std2.huge 140 32768 37 16
   nova flavor-create disk.huge 141 30720 880 8
   nova flavor-create mem.large.2 142 32768 8 8
   nova flavor-create mem.large.3 143 49152 8 8
   nova flavor-create mem.large.4 144 32768 8 4
   nova flavor-create cpu.medium.2 145 8192 8 8
   nova flavor-create mem.large.5 146 24576 8 4
   nova flavor-create disk.large.2 147 16384 320 8
   nova flavor-create mem.large.6 148 24576 16 4
   nova flavor-create std3.medium 149 8192 24 4
   nova flavor-create disk.hugito 150 28672 880 8
   nova flavor-create cpu.medium.3 151 4096 8 8
   nova flavor-create m1.small 2 2048 10 1
   nova flavor-create cfv 99 8192 3 8


* Instalar NewRelic::

   touch /etc/apt/sources.list.d/newrelic.list
   echo "deb http://apt.newrelic.com/debian/ newrelic non-free" >> /etc/apt/sources.list.d/newrelic.list
   wget -O- http://download.newrelic.com/548C16BF.gpg | apt-key add -
   apt-get update -y
   apt-get install newrelic-sysmond -y

* Clonar repositorio de cloud-host::

   apt-get install git -y
   /usr/bin/git clone -b master http://200.32.121.72/git/cloud-hosts /root/cloud-hosts


8. Nagios
=========================

* Add the controller to Nagios::

   IP=$(hostname -i)
   ssh -o StrictHostKeyChecking=no -i /root/.ssh/nagios.key root@$NAGIOS_HOST "if ! grep -i $(hostname) /usr/local/nagios/etc/objects/hosts/cloud.cfg >/dev/null; then
     echo \"define host {
           use                     linux-server
           host_name               $(hostname | tr -s  '[:lower:]'  '[:upper:]')
           alias                   $(hostname | tr -s  '[:lower:]'  '[:upper:]')
           address                 $IP
     }
   \" >> /usr/local/nagios/etc/objects/hosts/cloud.cfg
     /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
     /etc/init.d/nagios restart
   fi"



1. Adding a compute node
=========================

1.1. Preparing the Node
------------------

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

* Install ntp service::

   apt-get install ntp

* Configure the NTP server to follow the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server 10.111.82.1/g' /etc/ntp.conf
   service ntp restart  

* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
   sysctl -p

* Add this script to /etc/network/if-pre-up.d/iptablesload to forward traffic to em1::

   #!/bin/sh
   iptables -t nat -A POSTROUTING -o em1 -j MASQUERADE
   exit 0

1.2.Networking
------------

* Take a look at the networking::
   
   auto lo
   iface lo inet loopback

   auto em1
   iface em1 inet static
   address 10.111.82.2
   netmask 255.255.255.0
   network 10.111.82.0
   broadcast 10.111.82.255
   gateway 10.111.82.254
   dns-nameservers 10.1.1.68 10.1.1.42
   dns-search despexds.net

1.3 KVM
------------------

* Make sure that your hardware enables virtualization::

   apt-get install cpu-checker
   kvm-ok

* Normally you would get a good response. Now, move to install kvm and configure it::

   apt-get install -y kvm libvirt-bin pm-utils

* Restart the libvirt service::

   service libvirt-bin restart

1.4. Nova
------------------

* Install nova's required components for the compute node::

   apt-get install nova-compute nova-network nova-api-metadata

* Modify the /etc/nova/nova.conf like this::

   [DEFAULT]
   
   # LOGS/STATE
   verbose=True
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   
   # AUTHENTICATION
   auth_strategy=keystone
   
   # SCHEDULER
   scheduler_driver=nova.scheduler.multi.MultiScheduler
   compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
   
   # CINDER
   volume_api_class=nova.volume.cinder.API
   
   # DATABASE
   sql_connection=mysql://novaUser:novaPass@10.111.82.1/nova
   
   # COMPUTE
   libvirt_type=kvm
   libvirt_use_virtio_for_bridges=True
   start_guests_on_host_boot=True
   resume_guests_state_on_host_boot=True
   api_paste_config=/etc/nova/api-paste.ini
   allow_admin_api=True
   use_deprecated_auth=False
   nova_url=http://10.111.82.1:8774/v1.1/
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
   
   # APIS
   ec2_host=10.111.82.1
   ec2_url=http://10.111.82.1:8773/services/Cloud
   keystone_ec2_url=http://10.111.82.1:5000/v2.0/ec2tokens
   s3_host=10.111.82.1
   cc_host=10.111.82.1
   
   # RABBITMQ
   rabbit_host=10.111.82.1
   
   # GLANCE
   image_service=nova.image.glance.GlanceImageService
   glance_api_servers=10.111.82.1:9292
   
   # NETWORK
   network_manager=nova.network.manager.FlatDHCPManager
   force_dhcp_release=True
   dhcpbridge_flagfile=/etc/nova/nova.conf
   dhcpbridge=/usr/bin/nova-dhcpbridge
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   public_interface=em1
   flat_interface=em2
   flat_network_bridge=br100
   fixed_range=192.168.6.0/24
   network_size=256
   flat_network_dhcp_start=192.168.6.0
   flat_injected=False
   connection_type=libvirt
   multi_host=True
   
* Restart nova-* services::

  cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

2. Your First VM
============

To start your first VM:

* Create the master key pair::

   ssh-keygen -t dsa
   cp /root/.ssh/id_dsa.pub /root/master.pem
   nova keypair-add --pub-key /root/.ssh/id_dsa.pub master

* Find the ID from the image to boot::

   glance image-list

* Launch the instance using that ID::

   nova boot --image fb42188e-adce-4386-bc8c-99472033d525 --flavor m1.small --key-name master test --meta host=$(hostname)
