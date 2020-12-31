# okd-install - CURRENTLY UNDER CONSTRUCTION
OpenShift is hot, and while it is possible to use Red Hat Code Ready Containers (CRC), it's more appealing to set up a cluster with the upstream Open Source software, offered by the OKD project. A real cluster has multiple nodes, and easily gets you to require 64GB of RAM, or even much more. A single node cluster can also offer all of that functionality, and allows you to run OpenShift on your laptop, if it has 32GB of RAM or more. This readme will show you what is involved in installing a single-node OKD cluster.

# Required software
Let's face it, installation of OKD doesn't go smooth, and behavior will be different if you're using a different version of the software. In this procedure, the following components are used:
•	Physical computer: Fedora Workstation 33
•	okd-services: Fedora Workstation 33
•	okd-bootstrap: Fedora CoreOS 32 1104
•	okd-control: Fedora CoreOS 32 1104
•	openshift client: version 4.5.0
•	openshift installer: version 4.5.0
Notice that while writing this, more recent versions were available of the Fedora CoreOS image, as well as the openshift software. These versions however did present issues and for that reason are avoided here. 

# Machine configuration
OKD runs on Fedora CoreOS. In general, you don't install Fedora CoreOS (FCOS), you'll generate an ignition file on the node that we'll call the okd-services node. Next, you'll present this ignition file on a web server, and boot CoreOS from the ISO file. From there, you'll run the coreos-installer program to fetch the ignition file, which has been generated to automatically complete the setup for you.

In order to install OKD, you'll need two different FCOS instances. The first one is the bootstrap node (okd-bootstrap in this article). On this node you'll run a temporary OKD instance, which is used for the further configuration of the control node (okd-control in this article). Apart from that, you'll need the okd-services node to provide required services for setting up OKD.

The recommended minimal requirements to get you through the setup are as follows:

•	okd-service; 8GB RAM, 4 CPU cores, 40GB disk
•	okd-bootstrap; 8GB RAM, 4 CPU cores, 100GB disk
•	okd-control; 16GB RAM, 4 CPU cores, 120GB disk

Also, in this configuration I'm using some hard-coded IP addresses. Change as needed to match your configuration:
* okd-service: 192.168.122.40
* okd-bootstrap: 192.168.122.42
* okd-control: 192.168.122.41

Before you get started, it makes sense to understand the OKD installation procedure. OKD can be automatically provisioned on an existing infrastructure, which is what happens when it is deployed in public cloud, OpenStack, or VMware vSphere. The alternative is to provide it on user provided infrastructure (upi). This is the most flexible approach, where you'll be able to use your own virtual or physical machines. In this article I will describe how to install OKD on a KVM-based self-provided infrastructure. 

The okd-services node is used for different purposes throughout the installation:
•	Provides DNS services
•	Provides Proxy services
•	Runs a web service
•	Is used to generate the OKD ignition files
If you want to configure it even more smoothly, the okd-services node can also be set up with TFTP and DHCP services to offer PXE boot services so that the installation of the FCOS nodes can happen in a more elegant way. In this article we'll keep it as simple as possible and skip these additional services. 

#Configure DNS
1.  sudo dnf install -y bind bind-utils
2.  sudo cp named.conf /etc/
3.  sudo cp named.conf.local /etc/named/
4.  sudo mkdir /etc/named/zones
5.  sudo cp db* /etc/named/zones/
6.  sudo systemctl enable --now named
7.  sudo nmcli con mod <your-connection-name-here> ipv4.dns "127.0.0.1"

#Configure HAProxy
1.  sudo dnf install -y haproxy
2.  sudo cp haproxy.cfg /etc/haproxy/
3.  sudo ystemctl enable --now haproxy

#Install httpd
1.  sudo dnf install -y httpd
2.  sudo vim /etc/httpd/conf/httpd.conf # change Listen 80 to Listen 8080

#Open ports in firewall
1.  sudo firewall-cmd --add-port 8080/tcp --permanent
2.  sudo firewall-cmd --add-service http --permanent
3.  sudo firewall-cmd --add-service https --permanent
4.  sudo firewall-cmd --add-service dns --permanent
5.  sudo firewall-cmd --add-port 6443/tcp --permanent
6.  sudo firewall-cmd --add-port 22643/tcp --permanent
7.  sudo firewall-cmd --reload

#Additional config
1.  sudo setsebool -P httpd_read_user_content 1
2.  sudo setsebool -P haproxy_connect_any 1

#Download required OKD software
In this procedure, specific versions of the software are used. I've seen many different issues in different versions, so use other versions at your own risk. The versions mentioned here are tested and verified. make sure to isntall the following software:

Fedora CoreOS: fedora-coreos-32.202001104.3.0-live.x86_64.iso
openshift-client-linux-4.6.0*.tar.gz
openshift-install-linux-4.6.0*.tar.gz

Use tar to extract the tar balls and next use
sudo cp oc openshift-install kubectl /usr/local/bin

#generate the OKD installer files
1.  ssh-keygen # do NOT set an passphrase
2.  mkdir install_dir
3.  echo .ssh/id_rsa.pub >> install-config.yaml # make sure this is the yaml file downloaded from this git repo
4.  edit the yaml file to replace the SSH key placeholder with your SSH key, so not change the indentation!
5.  cp install-config.yaml install_dir/
6.  openshift-install create manifests --dir=install_dir/
7.  openshift-install create ignition-configs --dir=install_dir/
8.  sudo mkdir /var/www/html/okd
9.  sudo cp -R install_dir/ /var/www/html/okd/
10. sudo chown -R apache /var/www/html
11. sudo chmod -R 755 /var/www/html/okd4

#starting the bootstrap node
At this point, create the bootstrap node in KVM, using the following specs:
name: okd-bootstrap
RAM: 8192 MB
Disk: 100 GB
CPUs: 4

Boot the bootstrap node until you are in the coreos installation shell. 
From the shell, use sudo nmtui, and hard-code the IP address configuration:
IP address: 192.168.122.42/24
Gateway: 192.168.122.1
DNS: 192.168.122.40
Hostname: okd-bootstrap.example.com

Next, start the ignition process, using the following:
1.  sudo coreos-installer install /dev/vda --copy-network --ingition-url=http://okd-services.example.com:8080/okd/bootstrap.ign --insecure-ignition
Wait until this procedure is finished, then use sudo reboot to restart the bootstrap node. Once restarted, go have a cup of cofee and give it at least 5 minutes to do what it needs to do. 

To follow what is happening, use the following commands:
1.  ssh core@okd-bootstrap
2.  journalctl -b -f -u release-image.service -u bootkube.service

Don't get nervous if you see any error messages, but go have another cup of coffee, or start preparing the control node.

#starting the control node
Note: the software itself speaks about "master node". To avoid using sensitive terminology, I'm referring to this node as the control node. 

The control node setup needs the bootstrap node to complete its configuration. 
At this point, create the bootstrap node in KVM, using the following specs:
name: okd-bootstrap
RAM: 16384 MB
Disk: 100 GB
CPUs: 4

Boot the bootstrap node until you are in the coreos installation shell. 
From the shell, use sudo nmtui, and hard-code the IP address configuration:
IP address: 192.168.122.41/24
Gateway: 192.168.122.1
DNS: 192.168.122.40
Hostname: okd-control.example.com

Next, start the ignition process, using the following:
1.  sudo coreos-installer install /dev/vda --copy-network --ingition-url=http://okd-services.example.com:8080/okd/master.ign --insecure-ignition
Wait until this procedure is finished, then use sudo reboot to restart the bootstrap node. Once restarted, go have a cup of cofee and give it at least 5 minutes to do what it needs to do. 

Once it is running, you can use the following commands to verify how things are going:
1.  openshift-install wait-for bootstrap-complete
2.  openshift-install wait-for install-complete
3.  export KUBECONFIG=auth/kubeconfig
4.  watch -n5 'oc get co'

When the last command shows that all controllers are in an active state, your cluster is ready!

