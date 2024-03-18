# CDAC HPCSA Case Study 2022
### RAJA KANWAR 220940127051
## Overview
Build a two node Disk-less HPC-cluster using OpenHPC with Warewulf, Slurm, Nagios, and Ganglia and do a HPL benchmarking.

## Table of Contents
1. [Warewulf](#Warewulf) - Operating System provisioning platform for Linux
2. [Slurm](#Slurm) - Open-Source Workload Manager/Scheduler
3. [Nagios](#Nagios) - Open Source Infrastructure Monitoring Package
4. [Ganglia](#Ganglia) - calable Distributed System Monitoring Tool

5. [HPL-Benchmark](#HPL-Benchmark) - High-Performance Linpack benchmark implementation

## Project Structure
**The master node serves as the overall system management server (SMS) and is provisioned with CentOS7.7 and is subsequently configured to provision the remaining compute nodes with Warewulf in a stateless configuration.**
**For file systems, the master server will host an NFS file system that is made available to the compute nodes**
### Requirements :
#### VMware Require :
One Master Vm with Two Network Adapters
 

ens33 (NAT)
ens34 (Host-only)
 

Processor : 4
 

Ram : 8 GB
 

Storage : 100 GB
 
## Installation, Setup and Configuration
### 1. Pre-requisite:
We have to stop and disable firewall and disable selinux
- systemctl stop firewalld
- systemctl disable firewalld
- vi /etc/selinnux/conf
#sethostname of machine as master
- hostnamectl set-hostname master

### 2. Install OpenHPC Components
- yum install http://build.openhpc.community/OpenHPC:/1.3/CentOS_7/x86_64/ohpc-release-1.3-1.el7.x86_64.rpm
- yum repolistyum -y 
- install ohpc-baseyum -y 
- install ohpc-warewulfyum -y 
- install chronyvi /etc/chrony.conf
Edit this Conf. file -> server 192.168.149.147iburst,
allow 192.168.149.0/24 (uncomment and edit network address),
local stratum 10 (uncomment),
SAVE and Exit
- systemctl start chronyd
- systemctl enable chronydyum 
- install ntpdate
- ntpdate -q 192.168.149.147
- vi /etc/warewulf/provision.conf
edit -> change network device = ens34
- grep device /etc/warewulf/provision.conf
- vi /etc/xinetd.d/tftp
edit -> disable = no
- grep disable /etc/xinetd.d/tftp

### 3. Add Slurm services on master node
- yum -y install ohpc-slurm-server
- yum -y install slurm-sview-ohpc slurm-torque-ohpc
- vi /etc/slurm/slurm.conf
edit -> ClusterName=pearl,
-> ControlMachine=master,
-> NodeName=c[1-2], 
-> Nodes=c[1-2],                                                   --> This is my nodename
- grep NodeName= /etc/slurm/slurm.conf

- echo ens34ifconfig ens34
- systemctl restart xinetd
- systemctl enable mariadb.service
- systemctl restart mariadb
- systemctl enable httpd.service
- systemctl restart httpd
- systemctl enable dhcpd.service
- echo ${CHROOT}
- export CHROOT=/opt/ohpc/admin/images/centos7.7
- echo ${CHROOT}
- wwmkchroot centos-7$CHROOT                                           ->  Building initial BOS image
- uname -r
- chroot ${CHROOT} uname -r
- yum -y --installroot=${CHROOT} update
- yum -y --installroot=${CHROOT} install \

- ohpc-base-compute kernel kernel-headers kernel-devel kernel-tools parted \
- xfsprogs python-devel yum htop ipmitool glibc* perl perl-CPAN perl-CPAN \
- sysstat gcc make xauth firefox squashfs-tools
- cat /etc/resolv.conf
- vi /etc/resolv.conf ->add - master 192.168.149.147
- cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf
- yum -y --installroot=${CHROOT} install ohpc-slurm-client
- chroot ${CHROOT} systemctl enable slurmd
- yum -y --installroot=$CHROOT install chrony
- yum -y --installroot=$CHROOT install kernel lmod-ohpc
#Initialize warewulf database and ssh_keys
- wwinit database
- wwinit ssh_keys
- df -hT | grep -v tmpfs
- cat ${CHROOT}/etc/fstab
- echo "master:/home /home nfs nfsvers=3,nodev,nosuid 0 0" >> $CHROOT/etc/fstab
- echo "master:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=3,nodev 0 0" >> $CHROOT/etc/fstab
- cat ${CHROOT}/etc/fstab
- cat /etc/exports
- echo "/home *(rw,no_subtree_check,fsid=10,no_root_squash)" >> /etc/exports
- echo "/opt/ohpc/pub *(ro,no_subtree_check,fsid=11)" >> /etc/exports
- cat /etc/exports
- systemctl start nfs-server
- systemctl status nfs-server
- systemctl enable nfs-server
- exportfs -arv

- chroot $CHROOT systemctl enable chronyd
- echo "server 192.168.149.147 iburst" >> $CHROOT/etc/chrony.conf

## 4. Add Ganglia monitoring
- yum -y install ohpc-ganglia				->	# Install Ganglia meta-package on master
- yum -y --installroot=${CHROOT} install ganglia-gmond-ohpc			->	Install Ganglia compute node daemon
#Use example configuration script to enable unicast receiver on master host
- cp /opt/ohpc/pub/examples/ganglia/gmond.conf /etc/ganglia/gmond.conf    -> yes
- grep 'host =' /etc/ganglia/gmond.conf
- sed -i "s/<sms>/master/" /etc/ganglia/gmond.conf
- grep 'host =' /etc/ganglia/gmond.conf
- grep OpenHPC /etc/ganglia/gmond.conf
- sed -i "s/OpenHPC/carbon/" /etc/ganglia/gmond.conf
- grep carbon /etc/ganglia/gmond.conf

- cp /etc/ganglia/gmond.conf $CHROOT/etc/ganglia/gmond.conf      -> yes 
- echo "gridname carbon" >> /etc/ganglia/gmetad.conf
- grep gridname /etc/ganglia/gmetad.conf
- echo "systemctl enable gmond
- systemctl enable gmetad
- systemctl start gmond
- systemctl status gmetad
- chroot ${CHROOT} systemctl enable gmond" > /tmp/start_ganglia_service.sh
- bash /tmp/start_ganglia_service.sh
- grep "^date.timezone =" /etc/php.ini
- echo "date.timezone = Asia/Kolkata" >> /etc/php.ini
- grep "^date.timezone =" /etc/php.ini
- systemctl try-restart httpd
- Go to browser :  http://master/ganglia

## 6. Add Nagios monitoring

- yum -y install ohpc-nagios   -> Install Nagios meta-package on master host
- yum -y --installroot=$CHROOT install nagios-plugins-all-ohpc nrpe-ohpc      -> Install plugins into compute node image
- chroot $CHROOT systemctl enable nrpe
- perl -pi -e "s/^allowed_hosts=/# allowed_hosts=/" $CHROOT/etc/nagios/nrpe.cfg
- echo "nrpe 5666/tcp # NRPE" >> $CHROOT/etc/services
- echo "nrpe : 192.168.149.147 : ALLOW" >> $CHROOT/etc/hosts.allow
- echo "nrpe : ALL : DENY" >> $CHROOT/etc/hosts.allow
- chroot $CHROOT /usr/sbin/useradd -c "NRPE user for the NRPE service" -d /var/run/nrpe \-r -g nrpe -s /sbin/nologin nrpe
- chroot $CHROOT /usr/sbin/groupadd -r nrpeConfigure remote services to test on compute nodes
- mv /etc/nagios/conf.d/services.cfg.example /etc/nagios/conf.d/services.cfg
- mv /etc/nagios/conf.d/hosts.cfg.example /etc/nagios/conf.d/hosts.cfg
- for ((i=0; i<2; i++)) ; do perl -pi -e "s/HOSTNAME$(($i+1))/${c[$i]}/ || s/HOST$(($i+1))_IP/${c_ip[$i]}/" /etc/nagios/conf.d/hosts.cfg; done
- perl -pi -e "s/ \/bin\/mail/ \/usr\/bin\/mailx/g" /etc/nagios/objects/commands.cfg
- perl -pi -e "s/nagios\@localhost/root\@master/" /etc/nagios/objects/contacts.cfg
- echo command[check_ssh]=/usr/lib64/nagios/plugins/check_ssh localhost >> $CHROOT/etc/nagios/nrpe.cfg
- htpasswd -bc /etc/nagios/passwd nagiosadmin nagios       -> username : nagiosadmin   |    password: nagios
- chkconfig nagios on
- vi /etc/nagios/conf.d/hosts.cfg   -> Add clients and hostname
- systemctl start nagios
- systemctl status nagios

- chmod u+s 'which ping'
- go to browser : http://master/nagios
    -> username : nagioradmin
    ->password : nagios

- wwsh file list
- wwsh file import /etc/passwd
- wwsh file import /etc/group
- wwsh file import /etc/shadow
- wwsh file list

- export WW_CONF=/etc/warewulf/bootstrap.conf
- echo "drivers += updates/kernel/" >> $WW_CONF
- echo "modprobe += ahci, nvme"           >> $WW_CONF
- echo "drivers += overlay" >> $WW_CONF

- wwbootstrap `uname -r`
- echo ${CHROOT}
- wwvnfs --chroot $CHROOT
or
- wwvnfs --chroot /opt/ohpc/admin/images/centos7.7
- wwsh vnfs list

- echo "GATEWAYDEV=ens36" > /tmp/network.wwsh
- wwsh -y file import /tmp/network.wwsh --name network
- wwsh -y file set network --path /etc/sysconfig/network --mode=0644 --uid=0

- wwsh node new c1
- route -n 
- wwsh node set c1 --netdev ens34 --ipaddr=192.168.150.60 --hwaddr=00:0C:29:EC:16:C2 --netmask=255.255.255.0 --gateway 192.168.149.2
- wwsh node new c2
- wwsh node set c2 --netdev ens34 --ipaddr=192.168.150.61 --hwaddr=00:0c:29:78:09:c8 --netmask=255.255.255.0 --gateway 192.168.149.2
- wwsh  node list

- wwsh -y provision set c1 --vnfs=centos7.7 --bootstrap=`uname -r` --files=dynamic_hosts,passwd,group,shadow,network
 

- wwsh -y provision set c2 --vnfs=centos7.7 --bootstrap=`uname -r` --files=dynamic_hosts,passwd,group,shadow,network
 

- systemctl restart dhcpd && wwsh pxe update
## 7. HPL-Benchmark
- yum install epel-release -y

- yum install atlas -y

- rpm -ql atlas

- wget https://netlib.org/benchmark/hpl/hpl-2.3.tar.gz
 
- mv hpl-2.3.tar.gz /root/Downloads/

- cd /root/Downloads

- tar -zxvf hpl-2.3.tar.gz
 
- cd hpl-2.3/
- cd setup

- vim Make.Linux_Intel64

- wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.4.tar.gz
 
- mv openmpi-4.1.4.tar.gz /root/Downloads/

- tar -xvf openmpi-4.1.4.tar.gz

- cd openmpi-4.1.4/
- ./configure --prefix=/opt/openmpi-4.1.4 --enable-orterun-prefix-by-default

- make -j  8
- make install
- echo $PATH
- export PATH=/opt/openmpi-4.1.4/bin/:$PATH
- mp <Press TAB KEY>
- export LD_LIBRARY_PATH=/opt/openmpi-4.1.4/bin:$LD_LIBRARY_PATH
- echo $LD_LIBRARY_PATH
- cd ~/Downloads/hpl-2.3/setup
- cp Make.Linux_PII_CBLAS /root/Downloads/hpl-2.3
- cd /root/Downloads/hpl-2.3/
- rpm -ql atlas
- vim Make.Linux_PII_CBLAS
-> edit   

 

   # ------------ HPL Directory Structure / HPL library -------------------
    TOPdir       = /root/Downloads/hpl-2.3
   # ------------ Message Passing library (MPI) ----------------------------
    MPdir        = /opt/openmpi-4.1.4

    MPlib        = $(MPdir)/lib/libmpi.so 

  # -------------Compilers / linkers - Optimization flags ----------------

    CC           = /usr/bin/gcc
    LINKER       = /usr/bin/gcc
 # ----------Linear Algebra library (BLAS or VSIPL) ---------------------
 LAlib = $(LAdir)/libsatlas.so.3 $(LAdir)/libtatlas.so.3
>><Escape Key> : wq

- ake arch=Linux_PII_CBLAS

- cd /root/Downloads/hpl-2.3/bin/Linux_PII_CBLAS/
 
- vi HPL.dat

- mpirun --allow-run-as-root -np 4 ./xhpl HPL.dat
