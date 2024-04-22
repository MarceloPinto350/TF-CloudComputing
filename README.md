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
 * Master: 192.168.100.100
 * Worker1: 192.168.100.101
 * Worker2: 192.168.100.102 ...
4. Configuração das placas de rede locais
 * editar o arquivo /etc/netplan/00-installer-config.yaml e deixá-lo conforme abaixo:
 
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses: [192.168.100.100/16]
  version: 2
```
* Para aplicar as configurações execute: $ sudo netplan apply

### Instalação e configuração do cluster:
1. Proceder a exclusão do snapd
 * $ sudo apt purge snapd
2. Conectar na VM via SSH, copiar e rodar o script disponível em (https://gist.github.com/ramonfontes/b0858679da30adbf0b79c1716ee3f660), cujo conteúdo segue a seguir:

```shell
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
```

Sugestão de criar o arquivo config-k8s.ini e executá-lo, conforme segue
 * $ nano config-k8s.ini (copiar o conteúdo acima para o arquivo e salvar, executando-o em seguida).
 * $ sudo sh config-k8s.ini

### Para inicializar o cluster kubernetes: (No Master)
* sudo kubeadm init --apiserver-advertise-address 192.168.0.14 --pod-network-cidr=192.168.0.0/16 --v=5  
* Executar os comandos após configurar o cluster para ter acesso com o usuário criado na VM
```shell 
 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Adicionando workers na configuração do cluster kubernetes
1. Executar as configurações indicadas nos passos 1, 2 e 3, observando-se as configurações de rede indicadas para cada nó (worker)
2. Habilitar o acesso à rede interna do nó worker, para o master e demais nós, conforme o caso, conectar e sair (somente para confiar na chave), conforme o exemplo abaixo:
 * tfworker1:  $ ssh 192.168.100.100
  * tfmaster:  $ exit
 2.1.  Conectar o nó Master ao nó worker, conforme o exemplo a seguir:
  * tfmaster:  $ ssh 192.168.100.101
   * tfworker1: $ exit
 2.2. Conforme o caso, conectar o worker1 com o worker2 e vice-versa:
  * tfworker1: $ ssh 192.168.100.102
  * tfworker2: $ ssh 192.168.100.101
3. Adicionar o nó no Cluster, executando o comando abaixo:
 ```shell
 sudo kubeadm join 192.168.0.14:6443 --token 4d0jq2.ipdjevtb2x5fpq4a \
	--discovery-token-ca-cert-hash sha256:4c730d1e07fd377397f572e17856d2735a919576396e4ac42f6a4f69a9775cb1
```

## Reconfiguração do cluster
1. Importar o appliance;
2. Caso apresente erro simlar a este:
```log
$ kubectl get nodes -o wide
E0421 14:48:26.381095    2461 memcache.go:265] couldn't get current server API group list: Get "https://192.168.100.10:6443/api?timeout=32s": dial tcp 192.168.100.10:6443: connect: connection refused
E0421 14:48:26.381318    2461 memcache.go:265] couldn't get current server API group list: Get "https://192.168.100.10:6443/api?timeout=32s": dial tcp 192.168.100.10:6443: connect: connection refused
E0421 14:48:26.382570    2461 memcache.go:265] couldn't get current server API group list: Get "https://192.168.100.10:6443/api?timeout=32s": dial tcp 192.168.100.10:6443: connect: connection refused
E0421 14:48:26.382713    2461 memcache.go:265] couldn't get current server API group list: Get "https://192.168.100.10:6443/api?timeout=32s": dial tcp 192.168.100.10:6443: connect: connection refused
E0421 14:48:26.383987    2461 memcache.go:265] couldn't get current server API group list: Get "https://192.168.100.10:6443/api?timeout=32s": dial tcp 192.168.100.10:6443: connect: connection refused
The connection to the server 192.168.100.10:6443 was refused - did you specify the right host or port?
kadmin@tfmaster:~$ 
```
3. Executar os passos do script a seguir no **Master**:
```shell
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf /etc/kubernetes
sudo rm -rf /opt/cni
rm -rf .kube
sudo sh config-k8s.ini
# execute o comando abaixo trocando o endereço de rede pelo que estiver em modo NAT ou BRIDGE
sudo kubeadm init --apiserver-advertise-address 192.168.0.14 --pod-network-cidr=192.168.0.0/16 --v=5
# após a conclusão da configuração
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# copiar o trecho equivalente para inlcuir os workers no cluster
#kubeadm join <url-bridge ou nat:6443 --token al4za2.k0jq3o4pyu3k7vtx \
#	--discovery-token-ca-cert-hash sha256:96f286aff92c45ac7c7cc843f9d5ea86f513666ea5e9868a901aa634e8f64095


# Para verificar se está up
kubectl get nodes -o wide
```

4. Executar os passos do script abaixo nos nós **Worker** que desejar adicionar
```shell
sudo rm -rf /etc/kubernetes
sudo rm -rf /opt/cni
sudo sh config-k8s.ini
# executar o comando abaixo obtido da execução no master para adicionar o nó ao cluster
sudo kubeadm join <url-bridge ou nat>:6443 --token al4za2.k0jq3o4pyu3k7vtx \
	--discovery-token-ca-cert-hash sha256:96f286aff92c45ac7c7cc843f9d5ea86f513666ea5e9868a901aa634e8f64095
```
5. Para verificar se os nós foram adicionados no cluster, execute o comando abaixo no **Master**.
* $ kubectl get nodes -o wide


