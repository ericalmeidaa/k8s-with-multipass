# 🚀 Kubernetes Local Production-Grade Lab

Este repositório documenta a criação de um cluster Kubernetes local do zero, simulando um ambiente real de produção. A arquitetura utiliza ferramentas modernas de mercado para provisionamento, redes (eBPF), roteamento (Gateway API) e balanceamento de carga.

## 🏗️ Arquitetura e Stack de Tecnologias

*   **Virtualização:** Multipass (Ubuntu 24.04 LTS)
*   **Container Runtime:** containerd
*   **Kubernetes:** v1.30 (`kubeadm`, `kubelet`, `kubectl`)
*   **CNI (Rede):** Cilium (baseado em eBPF)
*   **Gateway / Ingress:** NGINX Gateway Fabric v1.3.0 + K8s Gateway API v1.1.0
*   **Load Balancer:** MetalLB v0.14.8
*   **Gerenciador de Pacotes:** Helm v3

---

## 📋 Pré-requisitos

*   Sistema Operacional host: Linux (Ubuntu recomendado), macOS ou Windows.
*   Recursos recomendados: Min. 4 CPUs e 12GB de RAM disponíveis.
*   [Multipass](https://multipass.run/) instalado (`sudo snap install multipass`).

---

## 🚀 Passo a Passo de Instalação

### 1. Provisionamento das Máquinas Virtuais (Nós)

Crie as VMs usando o Multipass. Neste lab, criamos 1 Control Plane e 2 Workers.

```bash
multipass launch 24.04 --name k8s-control-plane --cpus 2 --memory 4G --disk 20G
multipass launch 24.04 --name k8s-worker-1 --cpus 2 --memory 4G --disk 20G
multipass launch 24.04 --name k8s-worker-2 --cpus 2 --memory 4G --disk 20G
```
Acesso: Entre nas máquinas usando multipass shell <nome-da-vm> e anote o IP do Control Plane rodando ip a ou multipass ls no seu host.

### 2. Preparação do Sistema Operacional (Execute nos 3 Nós)

Acesse cada uma das VMs criadas e execute o script abaixo para configurar o sistema, instalar o containerd e as ferramentas do K8s (travadas na versão 1.30):

```bash
# Desativar Swap
sudo swapoff -a

# Configurar módulos do Kernel
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configurar parâmetros de rede
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Instalar dependências e Containerd
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg containerd

# Configurar Containerd para usar Systemd (obrigatório)
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Adicionar repositório oficial K8s e instalar ferramentas
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3. Inicialização do Cluster (Apenas no Control Plane)

No Control Plane, inicie o cluster substituindo <IP_DO_CONTROL_PLANE> pelo IP real da VM:

```Bash
sudo kubeadm init --apiserver-advertise-address=<IP_DO_CONTROL_PLANE>
```

Configure o acesso ao kubectl para o seu usuário normal:

```Bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g)$HOME/.kube/config
```

⚠️ Nota: No final do comando kubeadm init, será gerado o comando kubeadm join. Copie-o e execute com sudo nos nós Worker 1 e Worker 2 para anexá-los ao cluster.

### 4. Instalação da Rede / CNI (Cilium)

No Control Plane, instale o Cilium CLI e aplique no cluster:

```Bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
cilium install
```

Aguarde os Pods do Cilium ficarem prontos (cilium status --wait).

### 5. Configuração de Roteamento (NGINX Gateway API)

O Gateway API substitui o Ingress tradicional. (Execute no Control Plane):

```Bash
# 1. Instalar os CRDs genéricos
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# 2. Instalar os CRDs específicos do NGINX
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.3.0/deploy/crds.yaml

# 3. Instalar o Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 && chmod 700 get_helm.sh && ./get_helm.sh

# 4. Instalar o NGINX Gateway
helm install nginx-gateway oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace \
  --namespace nginx-gateway \
  --version 1.3.0
```

⚠️ Nota: Para permitir acesso via NodePort por qualquer IP do cluster sem restrições de nó, altere a política de tráfego com:
```bash
kubectl patch svc nginx-gateway-nginx-gateway-fabric -n nginx-gateway -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

### 6. Balanceador de Carga Local (MetalLB)

Remove o status **pending** do LoadBalancer entregando um IP real da sua rede local.

```Bash
# Instalar o MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s

# Criar o pool de IPs (Substitua a faixa conforme a rede das suas VMs)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.185.183.240-10.185.183.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF
```

### 7. Teste Prático (Aplicação Hello World)

Faça o deploy de uma aplicação web simples para validar o fluxo inteiro (Gateway > HTTPRoute > Service > Pods):

```Bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-route
  namespace: default
spec:
  parentRefs:
  - name: main-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: hello-service
      port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
  namespace: default
spec:
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: hashicorp/http-echo
        args:
        - "-text=Vitoria! Voce acessou um Pod rodando no Worker Node atraves do NGINX Gateway API!"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
EOF
```

### Validação

Execute:

```bash
kubectl get svc -n nginx-gateway 
```

Para descobrir o **EXTERNAL-IP** que o MetalLB atribuiu ao NGINX.

Acesse este IP diretamente no seu navegador.

Fim !!