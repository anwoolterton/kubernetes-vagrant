# -*- mode: ruby -*-
# vi: set ft=ruby :

# This Vagrant file requires the following plugins
# 1.

servers = [
  {
    :name => "k8s-master",
    :type => "master",
    :box => "ubuntu/bionic64",
    :box_version => "20200402.0.0",
    :mem => "2048",
    :cpu => "2"
  },
  {
    :name => "k8s-node-1",
    :type => "node",
    :box => "ubuntu/bionic64",
    :box_version => "20200402.0.0",
    :mem => "2048",
    :cpu => "2"
  },
  {
    :name => "k8s-node-2",
    :type => "node",
    :box => "ubuntu/bionic64",
    :box_version => "20200402.0.0",
    :mem => "2048",
    :cpu => "2"
  }
]

$configureBox = <<-SCRIPT

  apt-get install -y apt-transport-https ca-certificates curl software-properties-common

  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
  sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
  apt install -y kubelet kubeadm kubectl moreutils
  apt-mark hold kubelet kubeadm kubectl

  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
  
  apt-get install -y \
  containerd.io=1.2.13-1 \
  docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)

cat <<-EOF > /etc/docker/daemon.json
  {
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
  }
EOF

  mkdir -p /etc/systemd/system/docker.service.d

  systemctl daemon-reload
  systemctl restart docker
  usermod -aG docker vagrant

  apt-get update

  # kubelet requires swap off
  swapoff -a

  # keep swap off after reboot
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

  IP_ADDR=`ifdata -pa enp0s8`
  sudo sed -i "/^[^#]*KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=${IP_ADDR}" /etc/default/kubelet
  sudo systemctl restart kubelet
SCRIPT

$configureMaster = <<-SCRIPT
  echo "***** >>>> This is a MASTER <<<< *****"
  IP_ADDR=`ifdata -pa enp0s8`
  HOST_NAME=$(hostname -s)
  kubeadm init --apiserver-advertise-address=${IP_ADDR} --apiserver-cert-extra-sans=${IP_ADDR}  --node-name ${HOST_NAME} --pod-network-cidr=192.168.0.0/16

  #copying credentials to regular user - vagrant
  sudo --user=vagrant mkdir -p /home/vagrant/.kube
  cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

  # install Calico pod network addon
  export KUBECONFIG=/etc/kubernetes/admin.conf
  kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
  kubectl taint nodes --all node-role.kubernetes.io/master-

  kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
  chmod +x /etc/kubeadm_join_cmd.sh

  # required for setting up password less ssh between guest VMs
  sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
  sudo service sshd restart
SCRIPT

$configureNode = <<-SCRIPT
  echo "***** >>>> This is a WORKER <<<< *****"
  apt-get install -y sshpass
  sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@k8s-master:/etc/kubeadm_join_cmd.sh .
  sh ./kubeadm_join_cmd.sh
SCRIPT

Vagrant.configure("2") do |config|
  config.vbguest.auto_update = false
  config.hostmanager.manage_host = true
  ssh_info_public = true
  servers.each do |opts|
    config.vm.define opts[:name] do |config|

      config.vm.box = opts[:box]
      config.vm.box_version = opts[:box_version]
      config.vm.hostname = opts[:name]
      config.vm.network :public_network, bridge: 'en0: Wi-Fi (Wireless)'


      config.vm.provider "virtualbox" do |v|1
        v.name = opts[:name]
        v.customize ["modifyvm", :id, "--memory", opts[:mem]]
        v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
      end

      config.vm.provision "shell", inline: $configureBox
      if opts[:type] == "master"
        config.hostmanager.ip_resolver = proc do |machine|
          result = ''.dup
          machine.communicate.execute('/sbin/ifconfig enp0s8') do |type, data|
            result << data if type == :stdout
          end
          (ip = /inet (addr:)?(\d+\.\d+\.\d+\.\d+)/.match(result)) && ip[2]
          end
        config.vm.provision :hostmanager
        config.vm.provision "shell", inline: $configureMaster
      else
        config.hostmanager.ip_resolver = proc do |machine|
          result = ''.dup
          machine.communicate.execute('/sbin/ifconfig enp0s8') do |type, data|
            result << data if type == :stdout
          end
          (ip = /inet (addr:)?(\d+\.\d+\.\d+\.\d+)/.match(result)) && ip[2]
          end
        config.vm.provision :hostmanager
        config.vm.provision "shell", inline: $configureNode
      end
    end
  end
end 