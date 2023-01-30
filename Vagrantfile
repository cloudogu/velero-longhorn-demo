# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = 2
  end

  config.vm.synced_folder ".", "/vagrant", type: "rsync"

  # forward minio ports
  config.vm.network "forwarded_port", guest: 9000, host: 9000
  config.vm.network "forwarded_port", guest: 9090, host: 9090

  config.vm.provision "shell", inline: <<-SHELL
    # Add Helm APT repo
    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | tee /usr/share/keyrings/helm.gpg > /dev/null
    apt-get install apt-transport-https --yes
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | tee /etc/apt/sources.list.d/helm-stable-debian.list

    apt-get update
    apt-get install -y helm docker.io open-iscsi nfs-common

    # docker setup
    sudo groupadd docker
    sudo usermod -aG docker vagrant

    # Install K3s
    curl -sfL https://get.k3s.io | sh -
    # Configure kubeconfig
    mkdir -p /home/vagrant/.kube
    cp /etc/rancher/k3s/k3s.yaml /home/vagrant/.kube/config
    chown vagrant:vagrant /home/vagrant/.kube/config
    echo "export KUBECONFIG=~/.kube/config" >> /home/vagrant/.bashrc

    # Install K9s
    wget -c https://github.com/derailed/k9s/releases/download/v0.26.7/k9s_Linux_x86_64.tar.gz -O - | tar -xz -C /tmp/ && mv /tmp/k9s /usr/local/bin
  SHELL
end
