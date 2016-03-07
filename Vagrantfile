# using installation notes from
#    http://www.cloudera.com/content/www/en-us/documentation/enterprise/latest/topics/cdh_ig_cdh5_install.html
#
#
$master_script = <<SCRIPT
#!/bin/bash

yum install -y redhat-lsb-core curl

REPOCM=${REPOCM:-cm5}
CM_REPO_HOST=${CM_REPO_HOST:-archive.cloudera.com}
CM_MAJOR_VERSION=$(echo $REPOCM | sed -e 's/cm\\([0-9]\\).*/\\1/')
CM_VERSION=$(echo $REPOCM | sed -e 's/cm\\([0-9][0-9]*\\)/\\1/')
OS_CODENAME=$(lsb_release -sc)
OS_DISTID=$(lsb_release -si | tr '[A-Z]' '[a-z]')
if [ $CM_MAJOR_VERSION -ge 4 ]; then

set | grep ^CM
echo curl https://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/cloudera-cdh5.repo -o /etc/yum.repos.d/cloudera-cdh5.repo

curl https://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/cloudera-cdh5.repo -o /etc/yum.repos.d/cloudera-cdh5.repo

echo curl https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo -o /etc/yum.repos.d/cloudera-manager.repo

curl https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo -o /etc/yum.repos.d/cloudera-manager.repo

fi
yum -y clean all
yum -y update
yum -y install oracle-j2sdk1.7 bind-utils dnsmasq cloudera-manager-server-db-2 cloudera-manager-daemons cloudera-manager-server
#yum -y install oracle-j2sdk1.7 dnsmasq search hadoop-yarn-resourcemanager zookeeper zookeeper-server hadoop-hdfs-namenode hadoop-hdfs-secondarynamenode hadeeop-hdfs-datanode hadoop-mapreduce  hadoop-yarn-proxyserver hadoop-mapreduce-historyserver hadoop-client cloudera-manager-server-db-2 cloudera-manager-daemons cloudera-manager-server
#yum --enablerepo=epel install -y supervisor
service cloudera-scm-server-db initdb
service cloudera-scm-server-db start
service cloudera-scm-server start

SCRIPT

$hosts_script = <<SCRIPT
#!/bin/bash
#echo $master_script > master_script.tmp
#exit 0
cat > /etc/hosts <<EOF
127.0.0.1       localhost

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

EOF
SCRIPT

$slave_script = <<SCRIPT

#cat > /etc/dhcp/dhclient.conf <<EOF
#supersede domain-name-servers 192.168.254.100;
#EOF
#sudo dhclient

set | grep ^CM
echo curl https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo -o /etc/yum.repos.d/cloudera-manager.repo
curl https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo -o /etc/yum.repos.d/cloudera-manager.repo
fi

yum -y clean all
yum -y update


yum install -y bind-utils oracle-j2sdk1.7

SCRIPT

$optdisk_script = <<SCRIPT
#!/bin/bash

if [ -e /dev/sdb1 ]; then
  exit 0
fi

echo "opt_disk setup..."
yum install -y parted

# partition create
echo "part mkpart"
parted -s -a optimal /dev/sdb mklabel gpt -- mkpart primary ext4 0% 100%
# format filesystem
echo "ext4 format"
mkfs.ext4 -q /dev/sdb1
# mount disk
echo "mount disk"
mkdir /opt/cloudera
mount /dev/sdb1 /opt/cloudera
# fstab
echo "fstab addline"
echo `blkid /dev/sdb1 | awk '{print$2}' | sed -e 's/"//g'` /opt/cloudera ext4 defaults,relatime   0   0 >> /etc/fstab
#echo '/dev/sdb1 /opt/cloudera ext4 defaults,relatime 0 0' >> /etc/fstab

SCRIPT

Vagrant.configure("2") do |config|
  # Define base image
  config.vm.box = "vagrant-centos-6.7.box"
  #config.vm.box_url = "http://files.vagrantup.com/precise64.box"
  config.vm.box_url = "https://github.com/CommanderK5/packer-centos-template/releases/download/0.6.7/vagrant-centos-6.7.box"

  # Manage /etc/hosts on host and VMs
  config.hostmanager.enabled = false
  config.hostmanager.manage_host = true
  config.hostmanager.include_offline = true
  config.hostmanager.ignore_private_ip = false
  
  # Get disk path
  line = `VBoxManage list systemproperties | grep "Default machine folder"`
  vb_machine_folder = line.split(':')[1].strip()

  config.vm.define :master do |master|
    master.vm.provider :virtualbox do |v|
      v.name = "vm-cluster-node1"
      #v.customize ["modifyvm", :id, "--memory", "10240"]
      v.customize ["modifyvm", :id, "--memory", "6144"]

      second_disk = File.join(vb_machine_folder, v.name, 'disk2.vdi')

      storage_controller = 'SATA Controller'
      if not File.exist?(second_disk) then
        v.customize ["createhd",
                      "--filename", second_disk,
                      "--size", 30 * 1024] # 30 * 1024 = 30GB
      end
      v.customize ['storageattach', :id,
                    '--storagectl', storage_controller,
                    '--port', 1, #sdb
                    '--device', 0,
                    '--type', 'hdd',
                    '--medium', second_disk]
    end
    master.vm.network :private_network, ip: "192.168.254.100"
    master.vm.hostname = "vm-cluster-node1"
    master.vm.provision :shell, :inline => $hosts_script
    master.vm.provision :hostmanager
    master.vm.provision :shell, :inline => $optdisk_script
    master.vm.provision :shell, :inline => $master_script
  end

  config.vm.define :slave1 do |slave1|
    slave1.vm.box = "vagrant-centos-6.7.box"
    slave1.vm.provider :virtualbox do |v|
      v.name = "vm-cluster-node2"
      v.customize ["modifyvm", :id, "--memory", "2048"]
      #v.customize ["modifyvm", :id, "--memory", "3072"]
      second_disk = File.join(vb_machine_folder, v.name, 'disk2.vdi')

      storage_controller = 'SATA Controller'
      if not File.exist?(second_disk) then
        v.customize ["createhd",
                      "--filename", second_disk,
                      "--size", 30 * 1024] # 30 * 1024 = 30GB
      end
      v.customize ['storageattach', :id,
                    '--storagectl', storage_controller,
                    '--port', 1, #sdb
                    '--device', 0,
                    '--type', 'hdd',
                    '--medium', second_disk]
    end
    slave1.vm.network :private_network, ip: "192.168.254.101"
    slave1.vm.hostname = "vm-cluster-node2"
    slave1.vm.provision :shell, :inline => $hosts_script
    slave1.vm.provision :shell, :inline => $slave_script
    slave1.vm.provision :hostmanager
    slave1.vm.provision :shell, :inline => $optdisk_script
  end

  config.vm.define :slave2 do |slave2|
    slave2.vm.box = "vagrant-centos-6.7.box"
    slave2.vm.provider :virtualbox do |v|
      v.name = "vm-cluster-node3"
      v.customize ["modifyvm", :id, "--memory", "2048"]
      second_disk = File.join(vb_machine_folder, v.name, 'disk2.vdi')

      storage_controller = 'SATA Controller'
      if not File.exist?(second_disk) then
        v.customize ["createhd",
                      "--filename", second_disk,
                      "--size", 30 * 1024] # 30 * 1024 = 30GB
      end
      v.customize ['storageattach', :id,
                    '--storagectl', storage_controller,
                    '--port', 1, #sdb
                    '--device', 0,
                    '--type', 'hdd',
                    '--medium', second_disk]
    end
    slave2.vm.network :private_network, ip: "192.168.254.102"
    slave2.vm.hostname = "vm-cluster-node3"
    slave2.vm.provision :shell, :inline => $hosts_script
    slave2.vm.provision :shell, :inline => $slave_script
    slave2.vm.provision :hostmanager
    slave2.vm.provision :shell, :inline => $optdisk_script
  end

  config.vm.define :slave3 do |slave3|
    slave3.vm.box = "vagrant-centos-6.7.box"
    slave3.vm.provider :virtualbox do |v|
      v.name = "vm-cluster-node4"
      v.customize ["modifyvm", :id, "--memory", "2048"]
      second_disk = File.join(vb_machine_folder, v.name, 'disk2.vdi')

      storage_controller = 'SATA Controller'
      if not File.exist?(second_disk) then
        v.customize ["createhd",
                      "--filename", second_disk,
                      "--size", 30 * 1024] # 30 * 1024 = 30GB
      end
      v.customize ['storageattach', :id,
                    '--storagectl', storage_controller,
                    '--port', 1, #sdb
                    '--device', 0,
                    '--type', 'hdd',
                    '--medium', second_disk]
    end
    slave3.vm.network :private_network, ip: "192.168.254.103"
    slave3.vm.hostname = "vm-cluster-node4"
    slave3.vm.provision :shell, :inline => $hosts_script
    slave3.vm.provision :shell, :inline => $slave_script
    slave3.vm.provision :hostmanager
    slave3.vm.provision :shell, :inline => $optdisk_script
  end

end
