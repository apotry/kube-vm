# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
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

    yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes


    # ip of this box
    IP_ADDR=`ip address show eth1 | grep inet | head -n 1 | awk '{print $2}' | cut -f1 -d/`

    ARGS="--allow-privileged=true \
--fail-swap-on=false \
--enable-controller-attach-detach=false \
--container-log-max-size=100Mi \
--container-log-max-files=3 \
--node-ip=$IP_ADDR \
--kube-reserved-cgroup=/podruntime \
--system-reserved-cgroup=/system.slice \
--kubelet-cgroups=/podruntime/kubelet \
--runtime-cgroups=/podruntime/runtime \
--runtime-request-timeout=15m \
--container-runtime=remote \
--container-runtime-endpoint=unix:///run/containerd/containerd.sock \
--feature-gates=ExperimentalCriticalPodAnnotation=true \
--feature-gates=CRIContainerLogRotation=true \
--file-check-frequency=5s"

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
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16 --cri-socket=unix:///run/containerd/containerd.sock -v 10

    #copying credentials to regular user - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

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

Vagrant.configure("2") do |config|

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1] #, virtualbox__intnet: true

            config.vm.provider "virtualbox" do |v|
                v.name = opts[:name]
                v.customize ["modifyvm", :id, "--groups", "/ICD"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
                v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
            end

            config.vm.synced_folder ".kube", "/home/vagrant/.kube"

            config.vm.provision "file", source: "./calico.yaml", destination: "calico.yaml"
            config.vm.provision "file", source: "./calico-rbac.yaml", destination: "calico-rbac.yaml"
            config.vm.provision "file", source: "./config.toml", destination: "config.toml"

            config.vm.provision "shell", inline: $configureCommon

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
                # for i in 30000..32767
                #     config.vm.network :forwarded_port, guest: i, host: i
                # end
            end
        end
    end

    servers.each do |opts|

    end
end