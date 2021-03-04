VAGRANTFILE_API_VERSION = "2"

$install_misc = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|   misc section has been selected to install   |"
  echo    "-------------------------------------------------"
  echo    ""

  apt-get update
  apt-get -y install mc crudini glances
SCRIPT


$install_docker = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|   docker section has been selected to install   |"
  echo    "-------------------------------------------------"
  echo    ""

  apt-get update
  apt-get -y install apt-transport-https ca-certificates curl  gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

  apt-get update && apt-get -y install docker-ce=5:18.09.3~3-0~ubuntu-xenial docker-ce-cli=5:18.09.3~3-0~ubuntu-xenial
  systemctl start docker
SCRIPT


$configure_iptables = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|   iptables section has been selected to install |"
  echo    "-------------------------------------------------"
  echo    ""
  
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
  
SCRIPT


$install_kubeadm = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|   kubeadm section has been selected to install |"
  echo    "-------------------------------------------------"
  echo    ""

  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

  apt-get update
  apt-get install -y kubelet=1.19.0-00 kubeadm=1.19.0-00 kubectl=1.19.0-00
  apt-mark hold kubelet kubeadm kubectl

SCRIPT

$disable_swap = <<-SCRIPT
  echo    ""
  echo    "------------------------------------------------------------------"
  echo    "|   disable swap section has been selected to install             |"
  echo    "------------------------------------------------------------------"
  echo    ""
  
  swapoff -av
  sed -e '/swap/s/^/#/g' -i /etc/fstab

SCRIPT

$configure_hosts = <<-SCRIPT
  echo    ""
  echo    "------------------------------------------------------------------"
  echo    "|   configure hosts section has been selected to install          |"
  echo    "------------------------------------------------------------------"
  echo    ""
  
cat <<EOF | sudo tee /etc/hosts.new
172.16.94.10                    c1-master1
172.16.94.11                    c1-node1
172.16.94.12                    c1-node2
EOF
   
   sudo rm -fv /etc/hosts
   sudo mv -v /etc/hosts.new /etc/hosts
SCRIPT


$initialize_cluster = <<-SCRIPT
  echo    ""
  echo    "------------------------------------------------------------------"
  echo    "|   kubeadm initializatioin section has been selected to install |"
  echo    "------------------------------------------------------------------"
  echo    ""
  
  kubeadm init --apiserver-advertise-address=172.16.94.10 --pod-network-cidr=192.168.0.0/16

SCRIPT

$configure_kubectl = <<-SCRIPT
  echo    "------------------------------------------------------------------"
  echo    "| configure kubectl via copying config file to home directory     |"
  echo    "------------------------------------------------------------------"
  echo    ""

  mkdir -pv /home/vagrant/.kube
  sudo cp -v /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  sudo yes | cp -v /etc/kubernetes/admin.conf /vagrant/config
  sudo chown -v vagrant:vagrant /home/vagrant/.kube/config

SCRIPT

$write_token_and_hash = <<-SCRIPT
  echo    "------------------------------------------------------------------"
  echo    "| write kubernetes token and hash to /vagrant directory          |"
  echo    "------------------------------------------------------------------"
  echo    ""

  echo "token="$(kubeadm token list | tail -n 1 | awk '{print $1}') > /vagrant/credentials
  echo "hash="$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //') >> /vagrant/credentials
  
SCRIPT

$configure_flannel = <<-SCRIPT
  echo    "------------------------------------------------------------------"
  echo    "| configure flannel network on kubernetes master node             |"
  echo    "------------------------------------------------------------------"
  echo    ""

  export KUBECONFIG=/home/vagrant/.kube/config && kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  
SCRIPT

$configure_kubectl_node = <<-SCRIPT
  echo    "------------------------------------------------------------------"
  echo    "| configure kubectl via copying config file to home directory     |"
  echo    "------------------------------------------------------------------"
  echo    ""

  mkdir -pv /home/vagrant/.kube
  sudo cp -v /vagrant/config /home/vagrant/.kube/config
  sudo chown -v vagrant:vagrant /home/vagrant/.kube/config

SCRIPT

$join_node_to_cluster = <<-SCRIPT
  echo    "------------------------------------------------------------------"
  echo    "| join node to kubernetes cluster                                |"
  echo    "------------------------------------------------------------------"
  echo    ""

  token=$(crudini --get /vagrant/credentials '' token)
  hash=$(crudini --get /vagrant/credentials '' hash)
  
  echo $token
  echo $hash
  export KUBECONFIG=/home/vagrant/.kube/config && kubeadm join 172.16.94.10:6443 --token $token --discovery-token-ca-cert-hash sha256:$hash
  kubectl get nodes
  
SCRIPT

$upgrade_server = <<-SCRIPT
  echo    "------------------------------------------------------------------"
  echo    "| upgrade to the new version                                     |"
  echo    "------------------------------------------------------------------"
  echo    ""

  apt-cache madison kubeadm
  apt-mark unhold kubeadm
  apt-get update
  sudo apt-get install -y kubeadm=1.20.0-00
  apt-mark hold kubeadm
  export KUBECONFIG=/home/vagrant/.kube/config && kubectl get nodes
  export KUBECONFIG=/home/vagrant/.kube/config && kubectl drain c1-master1 --ignore-daemonsets
  export KUBECONFIG=/home/vagrant/.kube/config && kubectl get nodes
  export KUBECONFIG=/home/vagrant/.kube/config && kubectl kubeadm plan
  export KUBECONFIG=/home/vagrant/.kube/config && sudo kubeadm upgrade apply v1.20.0 -y
  export KUBECONFIG=/home/vagrant/.kube/config && sudo kubectl uncordon $node
  
SCRIPT

$upgrade_kubectl_kubelet = <<-SCRIPT
  node_name=$1
  version=$2
  #1.20.0-00
  echo    "------------------------------------------------------------------"
  echo    "| upgrade to the new version                                     |"
  echo    "------------------------------------------------------------------"
  echo    ""

  echo "node name is : "$node_name
  echo "version of kube adm/ctl is : "$version

  echo ""

  sudo apt-cache madison kubelet
  sudo apt-mark unhold kubelet kubectl
  sudo apt-get update
  sudo apt-get update && sudo apt-get install -y kubelet=$version kubectl=$version
  sudo apt-mark hold kubelet kubectl
  
  export KUBECONFIG=/home/vagrant/.kube/config && kubectl get nodes
  export KUBECONFIG=/home/vagrant/.kube/config && kubectl drain $node_name --ignore-daemonsets
  export KUBECONFIG=/home/vagrant/.kube/config && kubeadm upgrade node
  export KUBECONFIG=/home/vagrant/.kube/config && kubectl uncordon $node_name

  sudo systemctl daemon-reload
  sudo systemctl restart kubelet

  export KUBECONFIG=/home/vagrant/.kube/config && kubectl get nodes
  
SCRIPT


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "n1" do |n1|
    n1.vm.box = "bento/ubuntu-16.04"
    n1.vm.hostname = "c1-node1"
    n1.vm.network :private_network, ip: "172.16.94.11"
	
	config.vm.provision "misc", type: "shell" do |shell|
      shell.inline = $install_misc
    end
	
	config.vm.provision "docker", type: "shell" do |shell|
       shell.inline = $install_docker
    end
	
	config.vm.provision "iptables", type: "shell" do |shell|
       shell.inline = $configure_iptables
    end
	
	config.vm.provision "kubeadm", type: "shell" do |shell|
       shell.inline = $install_kubeadm
    end

	config.vm.provision "disable_swap", type: "shell" do |shell|
       shell.inline = $disable_swap
    end
	
	config.vm.provision "configure_hosts", type: "shell" do |shell|
       shell.inline = $configure_hosts
    end

  	config.vm.provision "configure_kubectl_node", type: "shell" do |shell|
       shell.inline = $configure_kubectl_node
    end

  	config.vm.provision "join_node_to_cluster", type: "shell" do |shell|
       shell.inline = $join_node_to_cluster
    end
	
	config.vm.provision "upgrade_kubectl_kubelet", type: "shell" do |shell|
       shell.inline = $upgrade_kubectl_kubelet
	   shell.args = ["c1-node1","1.20.0-00"]
    end
	
  end

  config.vm.define "n2" do |n2|
    n2.vm.box = "bento/ubuntu-16.04"
    n2.vm.hostname = "c1-node2"
    n2.vm.network :private_network, ip: "172.16.94.12"

	config.vm.provision "misc", type: "shell" do |shell|
      shell.inline = $install_misc
    end
	
	config.vm.provision "docker", type: "shell" do |shell|
       shell.inline = $install_docker
    end
	
	config.vm.provision "iptables", type: "shell" do |shell|
       shell.inline = $configure_iptables
    end
	
	config.vm.provision "kubeadm", type: "shell" do |shell|
       shell.inline = $install_kubeadm
    end

	config.vm.provision "disable_swap", type: "shell" do |shell|
       shell.inline = $disable_swap
    end
	
	config.vm.provision "configure_hosts", type: "shell" do |shell|
       shell.inline = $configure_hosts
    end

  	config.vm.provision "configure_kubectl_node", type: "shell" do |shell|
       shell.inline = $configure_kubectl_node
    end

  	config.vm.provision "join_node_to_cluster", type: "shell" do |shell|
       shell.inline = $join_node_to_cluster
    end

	config.vm.provision "upgrade_kubectl_kubelet", type: "shell" do |shell|
       shell.inline = $upgrade_kubectl_kubelet
       shell.args = ["c1-node2","1.20.0-00"]
    end
	
  end

  config.vm.define "km1" do |km1|
    km1.vm.box = "bento/ubuntu-16.04"
    km1.vm.hostname = "c1-master1"
    km1.vm.network :private_network, ip: "172.16.94.10"
	km1.vm.network "forwarded_port", guest: 8001, host: 8888

	config.vm.provision "misc", type: "shell" do |shell|
      shell.inline = $install_misc
      shell.args = ["no_args"]
    end
	
	config.vm.provision "docker", type: "shell" do |shell|
       shell.inline = $install_docker
    end
	
	config.vm.provision "iptables", type: "shell" do |shell|
       shell.inline = $configure_iptables
    end
	
	config.vm.provision "kubeadm", type: "shell" do |shell|
       shell.inline = $install_kubeadm
    end

	config.vm.provision "disable_swap", type: "shell" do |shell|
       shell.inline = $disable_swap
    end
	
	config.vm.provision "configure_hosts", type: "shell" do |shell|
       shell.inline = $configure_hosts
    end
	
	config.vm.provision "initialize_cluster", type: "shell" do |shell|
       shell.inline = $initialize_cluster
    end

  	config.vm.provision "configure_kubectl", type: "shell" do |shell|
       shell.inline = $configure_kubectl
    end

	config.vm.provision "write_token_and_hash", type: "shell" do |shell|
       shell.inline = $write_token_and_hash
    end	

	config.vm.provision "configure_flannel", type: "shell" do |shell|
       shell.inline = $configure_flannel
    end	
   end
   
  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "2024"]
    vb.customize ["modifyvm", :id, "--cpus", "2"]
  end
end
