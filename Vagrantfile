# -*- mode: ruby -*-
# vi: set ft=ruby :

masterName     = "centos-master"
masterIp       = "192.168.121.9"
noOfNodes      = 3
nodeName       = "centos-node-"
nodeNetIp      = "192.168.121."
nodeHostBaseIp = 65
nodeSubnet     = 65

Vagrant.configure(2) do |config|

  config.vm.box = "boxcutter/centos71"

  # set user to be root
  config.ssh.username = 'root'
  config.ssh.password = 'vagrant'
  config.ssh.insert_key = 'true'

  config.vm.provider "virtualbox" do |vb|
     # this line enables communication over host VPN connection by guest
     vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]

     vb.customize ["modifyvm", :id, "--memory", "1024"]
     vb.customize ["modifyvm", :id, "--cpus", "1"]
  end

  config.vm.provision "shell",
     args: [ masterName, masterIp, nodeName, nodeNetIp, nodeHostBaseIp, noOfNodes ],
     inline: <<-SHELL

  masterHostname="$1"
  masterIp="$2"
  nodeHostname="$3"
  nodeNetIp="$4"
  nodeHostBaseIp="$5"
  noOfNodes="$6"

  cat <<EOT >> /etc/yum.repos.d/virt7-docker-common-release.repo
[virt7-docker-common-release]
name=virt7-docker-common-release
baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
gpgcheck=0
EOT

  yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd

  echo "${masterIp}	${masterHostname}" >> /etc/hosts
  for (( node=1; node<=${noOfNodes}; node++ ))
  do
    hostIp = ${nodeHostBaseIp}
    echo "${nodeNetIp}$(($nodeHostBaseIp + $node - 1))	${nodeHostname}${node}" >> /etc/hosts;
  done

  cat <<EOT > /etc/kubernetes/config
# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://${masterHostname}:2379"

# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the replication controller and scheduler find the kube-apiserver
KUBE_MASTER="--master=http://${masterHostname}:8080"
EOT

  systemctl disable firewalld
  systemctl stop firewalld

  SHELL

  config.vm.define masterName do |master|
    master.vm.hostname = masterName
    master.vm.network :private_network, ip: masterIp

    master.vm.provision "shell", inline: <<-SHELL

    cat <<EOT > /etc/etcd/etcd.conf
# [member]
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"

#[cluster]
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
EOT

    cat <<EOT > /etc/kubernetes/apiserver
# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port kubelets listen on
KUBELET_PORT="--kubelet-port=10250"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# Add your own!
KUBE_API_ARGS=""
EOT

     for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do
       systemctl restart $SERVICES
       systemctl enable $SERVICES
       systemctl status $SERVICES
     done

   SHELL

  end

  (1..noOfNodes).each do |i|
    config.vm.define "#{nodeName}#{i}" do |node|

      node.vm.hostname = "#{nodeName}#{i}"
      node.vm.network :private_network, ip: "#{nodeNetIp}#{nodeSubnet}"
      nodeSubnet += 1

      node.vm.provision "shell",
             args: [ masterName, "#{nodeName}#{i}" ],
             inline: <<-SHELL

      masterHostname="$1"
      hostname="$2";

      cat <<EOT > /etc/kubernetes/kubelet
# The address for the info server to serve on
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=${hostname}"

# Location of the api-server
KUBELET_API_SERVER="--api-servers=http://${masterHostname}:8080"

# Add your own!
KUBELET_ARGS=""
EOT

      for SERVICES in kube-proxy kubelet docker; do
        systemctl restart $SERVICES
        systemctl enable $SERVICES
        systemctl status $SERVICES
      done

      SHELL

    end
  end

end
