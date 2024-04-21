# Trabalho Final da disciplina Cloud Computing


## Grupo composto
* Alykson
* Marcelo

## Configuração do Cluster

### Imagens da máquinas virtuais
   * [TFMaster.ova] (https://drive.google.com/file/d/1Ajt0okblPDFYCTTKkYNMu84nFazJt7nK/view?usp=drive_link)
   * [TFWorker1.ova] (https://drive.google.com/file/d/1aqz9p1eXgwlrwnCwWVpXxowzJg9uOulW/view?usp=drive_link)

### Preparação das máquinas do cluster
1. Instalar o Ubuntu Server 22.04 mais limpo possível, com o SSH server
2. Colocar a máquina com a placa de rede em modo  Bridge
3. Adicionar a segunda placa de rede nas VMs, conectado a: Rede Interna, com o Nome: knet, com os seguintes endereçamentos:
 * Master: 192.168.100.10
 * Worker1: 192.168.100.15
 * Worker2: 192.168.100.20
4. Configuração das placas de rede locais
 * editar o arquivo /etc/netplan/00-installer-config.yaml e deixá-lo conforme abaixo
 
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses: [10.100.0.10/24]
  version: 2
```

### Instalação e configuração do cluster:
1. Proceder a exclusão do snapd
 * $ sudo apt purge snapd
2. Conectar na VM via SSH, copiar e rodar o script disponível em (https://gist.github.com/ramonfontes/b0858679da30adbf0b79c1716ee3f660), cujo conteúdo segue a seguir:
`
#!/bin/bash 

export DEBIAN_FRONTEND=noninteractive

apt-get update
apt-get install -y git wget net-tools

# Prereqs
modprobe overlay
modprobe br_netfilter

swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

# Install Containerd as Runtime
apt-get install -yq \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    gnupg \
    software-properties-common
    
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" |  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install -yq containerd.io

containerd config default |  tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd

# Install kubeadm, kubectl and kubelet

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update -y
apt-get install -y  kubelet kubeadm kubectl kubernetes-cni nfs-common

# Additional Configuration
#sysctl net.bridge.bridge-nf-call-iptables=1

rm -rf /var/lib/kubelet/*
apt-get install nfs-common -y
`
Sugestão de de crir o arquivo config-k8s.ini e executá-lo, conforme segue
 * $ nano config-k8s.ini
 Compiar o conteúdo acima para o arquivo e salvar.
 * $ sudo sh config-k8s.ini

### Para inicializar o cluster kubernetes: (No Master)
 * $ sudo kubeadm init --apiserver-advertise-address 192.168.100.10 --pod-network-cidr=192.168.0.0/16 (com o endereço da 2ª placa) - ao reiniciar ele perde a referência e não roda o cluster
 sudo kubeadm init --apiserver-advertise-address 192.168.0.18 --pod-network-cidr=10.100.0.0/16 -v=5 (com o endereço da 1ª placa)
* Executar os comandos após configurar o cluster para ter acesso com o usuário criado na VM
 * $ mkdir -p $HOME/.kube
 * $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 * $ sudo chown $(id -u):$(id -g) $HOME/.kube/config


Configuração dos Nodes
Executar as configurações indicadas nos passos 1, 2 e 3, observando-se as configurações de rede indicadas para cada nó (worker)
Habilitar o acesso à rede interna do nó worker, para o master e demais nós, conforme o caso, conectar  sair (somente para confiar na chave), conforme o exemplo abaixo:
tfworker1 $ ssh 192.168.100.10
tfmaster $ exit
Conectar o nó Master ao nó worker, conforme o exemplo a seguir:
tfmaster $ ssh 192.168.0.101
tfworker1 $ exit
Conforme o caso, conectar o worker1 com o worker2 e vice-versa:
tfworker1 $ ssh 192.168.0.102
tfworker2 $ ssh 192.168.0.101
Adicionar o nó no Cluster, executando o comando abaixo:
sudo kubeadm join 192.168.0.18:6443 --token t2jfcw.dnkuklpn7pi27e15 \
	--discovery-token-ca-cert-hash sha256:b5f7c04eadf55cb193fc924bc8bd050136fdfd26352bc2641e008764c5e79d32




