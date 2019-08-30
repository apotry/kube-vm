# -*- mode: ruby -*-
# vi: set ft=ruby :

nodes = [
    {
        :name => "k8s-master",
        :type => "master",
        :box => "centos/7",
        :box_version => "1902.01",
        :eth1 => "192.168.205.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-1",
        :type => "node",
        :box => "centos/7",
        :box_version => "1902.01",
        :eth1 => "192.168.205.11",
        :mem => "6144",
        :cpu => "2"
    },
    {
        :name => "k8s-node-2",
        :type => "node",
        :box => "centos/7",
        :box_version => "1902.01",
        :eth1 => "192.168.205.12",
        :mem => "6144",
        :cpu => "2"
    },
    {
        :name => "k8s-node-3",
        :type => "node",
        :box => "centos/7",
        :box_version => "1902.01",
        :eth1 => "192.168.205.13",
        :mem => "6144",
        :cpu => "2"
    },
]

$configureCommon = <<-SCRIPT
    CONTAINERD_VERSION=1.2.4
    KUBERNETES_VERSION=1.15.3

    yum update
    yum install -y wget

    # kubelet requires swap off
    swapoff -a

    # Set SELinux in permissive mode (effectively disabling it)
    setenforce 0
    sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

    # Stop & disable firewalld
    systemctl stop firewalld
    systemctl disable firewalld

    # pre-requisites for calico
    modprobe overlay
    modprobe br_netfilter

    cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

    cat > /etc/sysctl.d/98-es-max-map-count.conf <<EOF
vm.max_map_count=1048576
EOF

    sysctl --system

    wget https://storage.googleapis.com/cri-containerd-release/cri-containerd-${CONTAINERD_VERSION}.linux-amd64.tar.gz
    tar xzf cri-containerd-${CONTAINERD_VERSION}.linux-amd64.tar.gz -C /
    mkdir -p /etc/containerd
    cp /home/vagrant/config.toml /etc/containerd/config.toml

    systemctl daemon-reload
    systemctl enable containerd && systemctl start containerd

    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

    yum install -y kubelet-$KUBERNETES_VERSION kubeadm-$KUBERNETES_VERSION kubectl-$KUBERNETES_VERSION --disableexcludes=kubernetes


    # ip of this box
    IP_ADDR=`ip address show eth1 | grep inet | head -n 1 | awk '{print $2}' | cut -f1 -d/`

    ARGS="--node-ip=$IP_ADDR --config=/home/vagrant/kubelet-conf.json"

    sed -i "s|KUBELET_EXTRA_ARGS=.*|KUBELET_EXTRA_ARGS=$ARGS|g" /etc/sysconfig/kubelet

    systemctl daemon-reload
    systemctl enable --now kubelet
SCRIPT

$configureMaster = <<-SCRIPT
    echo "This is master"
    # ip of this box
    IP_ADDR=`ip address show eth1 | grep inet | head -n 1 | awk '{print $2}' | cut -f1 -d/`

    echo $IP_ADDR

    # install k8s master
    HOST_NAME=$(hostname -s)
    kubeadm init --config=/home/vagrant/kubeadm-conf.yaml --node-name $HOST_NAME  --cri-socket=unix:///run/containerd/containerd.sock -v 10

    #copying credentials to regular user - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube

    cp -f /etc/kubernetes/admin.conf /home/vagrant/.kube/admin.conf
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/admin.conf

    export KUBECONFIG=/etc/kubernetes/admin.conf

    # install Calico pod network addon
    KEY=$(cat /etc/kubernetes/pki/etcd/server.key | base64 -w 0)
    CERT=$(cat /etc/kubernetes/pki/etcd/server.crt | base64 -w 0)
    CA=$(cat /etc/kubernetes/pki/etcd/ca.crt | base64 -w 0)

    sed -i "s/etcd-key: .*/etcd-key: $KEY/g" /home/vagrant/calico.yaml
    sed -i "s/etcd-cert: .*/etcd-cert: $CERT/g" /home/vagrant/calico.yaml
    sed -i "s/etcd-ca: .*/etcd-ca: $CA/g" /home/vagrant/calico.yaml
    sed -i "s|etcd_endpoints: .*|etcd_endpoints: https:\/\/$IP_ADDR:2379|g" /home/vagrant/calico.yaml

    kubectl apply -f /home/vagrant/calico-rbac.yaml
    kubectl apply -f /home/vagrant/calico.yaml

    # create Calico API Config file
    ## 1. copy the cert files
    cp /etc/kubernetes/pki/etcd/server.key /home/vagrant/.kube/etcd-key.pem
    cp /etc/kubernetes/pki/etcd/server.crt /home/vagrant/.kube/etcd-cert.pem
    cp /etc/kubernetes/pki/etcd/ca.crt /home/vagrant/.kube/etcd-ca.pem

    ## 2. create the CalicoAPIConfig yaml file
    cat > /home/vagrant/.kube/calico-api-config.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
spec:
  datastoreType: etcdv3
  etcdEndpoints: https://$IP_ADDR:2379
  etcdKeyFile: $1/.kube/etcd-key.pem
  etcdCertFile: $1/.kube/etcd-cert.pem
  etcdCACertFile: $1/.kube/etcd-ca.pem
EOF

    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh

    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart

SCRIPT

$configureNode = <<-SCRIPT
    yum -y install sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
    sh ./kubeadm_join_cmd.sh
    swapon /swapfile
SCRIPT

$removeMasterTaint = <<-SCRIPT
    export KUBECONFIG=/etc/kubernetes/admin.conf
    if [ "$1" == "master-only: true" ]; then
        kubectl taint node k8s-master node-role.kubernetes.io/master-
    fi
SCRIPT

$installMetricsServer = <<-SCRIPT
    export KUBECONFIG=/etc/kubernetes/admin.conf
    if [ "$1" == "install: true" ]; then
        kubectl create -f /vagrant/metrics-server/deploy/1.8+/
        kubectl patch deploy metrics-server -n kube-system --patch "$(cat /vagrant/patch-metrics-server.yaml)"
    fi
SCRIPT

Vagrant.configure("2") do |config|
    nodes.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]

            config.vm.provider "virtualbox" do |v|
                v.name = opts[:name]
                v.customize ["modifyvm", :id, "--groups", "/ICD"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
                v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
                v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
                v.customize ["modifyvm", :id, "--nictype1", "virtio"]
                v.customize ["modifyvm", :id, "--nictype2", "virtio"]

            end

            config.vm.provision "file", source: "./calico.yaml", destination: "calico.yaml"
            config.vm.provision "file", source: "./calico-rbac.yaml", destination: "calico-rbac.yaml"
            config.vm.provision "file", source: "./config.toml", destination: "config.toml"
            config.vm.provision "file", source: "./kubelet-conf.json", destination: "kubelet-conf.json"
            config.vm.provision "file", source: "./kubeadm-conf.yaml", destination: "kubeadm-conf.yaml"
            config.vm.provision "file", source: "./audit-policy.yaml", destination: "audit-policy.yaml"
            config.vm.synced_folder ".", "/vagrant", type: "rsync", rsync__args: ["--verbose", "--archive", "--delete", "-z"]

            config.vm.synced_folder "./oci", "/oci"

            config.vm.provision "shell", inline: $configureCommon

            if opts[:type] == "master"
                config.vm.synced_folder ".kube/", "/home/vagrant/.kube/"
                config.vm.provision "shell", inline: $configureMaster, args: [File.dirname(__FILE__)]
            else
                config.vm.provision "shell", inline: $configureNode
            end

            config.vm.provision "shell", inline: $removeMasterTaint, args: ["master-only: #{nodes.count == 1}"]
            config.vm.provision "shell", inline: $installMetricsServer, args: ["install: #{nodes.count == 1 || opts[:type] == 'node'}"]
        end
    end
end
