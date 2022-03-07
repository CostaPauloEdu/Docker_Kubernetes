# TCC

Desenvolvimento do projeto do TCC.


## INSTALAÇÃO DO DOCKER


Instalar o pacote "curl" para fazer as requisições HTTP/HTTPS: 

    apt install curl

O comando abaixo realizará o download do pacote de instalação do Docker e realizará a instalação através do arquivo "sh":

    curl -fsSL https://get.docker.com | sh

## INSTALAÇÃO DO GITLAB

Antes de instalar o GitLab devemos criar a variável "GITLAB_HOME" e apontar o caminho do mapeamento do volume do container:

	export GITLAB_HOME=/srv/gitlab
  
Abaixo é o comando para criação do container do GitLab:

	docker run --detach \
    --hostname 192.168.0.16 \
    --publish 8443:443 --publish 8484:80 --publish 2222:22 \  
    --name gitlab \
    --restart always \
    --volume $GITLAB_HOME/config:/etc/gitlab \
    --volume $GITLAB_HOME/logs:/var/log/gitlab \
    --volume $GITLAB_HOME/data:/var/opt/gitlab \
    --shm-size 256m \
    gitlab/gitlab-ce:latest

O comando a seguir deve ser executado apenas quando o container for criado. Ele irá gerar a senha inicial do usuário root para acesso ao GitLab.

    docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
	
Após um bom tempo o serviço web do GitLab é iniciado, deverá ser feito o login como root e a senha gerada através do comando anterior. Depois disso, deverá ser feito uma nova senha, visto que a senha inicial expira com 24h.  
	
## INSTALAÇÃO DO KUBERNETES

Para instalar o Kubernetes é necessário inicialmente ativar os filtros e configurações de rede:

    cat <<EOF | tee /etc/modules-load.d/k8s.conf
    br_netfilter
    EOF

No mesmo sentido do comando anterior, o próximo servirá para fazer um modo "bridge" nas placas de rede IPv4 e IPv6 do host do Cluster do Kubernetes com o serviço que será criado, a fim de se enxergarem na rede e também poder ter comunicação com a internet:

    cat <<EOF | tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    EOF

Para a instalação usando o Debian 11, é necessário inserir o diretório /sbin no PATH para que possamos executar alguns comandos que necessitam elevação para execução.

    export PATH=$PATH:/sbin

Devido ao serviço compartilhado de hardware que o Kubernetes fornece, é necessário desabilitar o SWAP:

    swapoff -a

Para aplicar as configurações sem a necessidade de dá um reboot na máquina, podemos usar o seguinte comando:

    sysctl --system


Agora iremos adicionar a chave pública para o serviço do Kubernetes:

    curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

Inserindo numa "apt list" o repositório oficial do Kubernetes para instalação das versões atualizadas:

    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list

Atualizando os repositórios:

    apt-get update

Instalando os serviços do Kubernetes (kubelet, kubeadm e kubectl)

    apt-get install -y kubelet kubeadm kubectl

Fazendo a marcação dos serviços recém instalados:

    apt-mark hold kubelet kubeadm kubectl

Para iniciar um novo Cluster de Kubernetes é necessário definir também a rede pela qual os PODs, Deployments e Serviços devem operar. Nesse contexto iremos aplicar o direcionamento para a rede 192.168.0.0/16 que é compatível com o POD de DNS do Calico:

    kubeadm init --pod-network-cidr=192.168.0.0/16

Após a inicialização do Cluster, execute os três comandos abaixo:

    mkdir -p $HOME/.kube
 
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 
    chown $(id -u):$(id -g) $HOME/.kube/config

Caso queira adicionar um novo "node" vinculado ao master que estamos criando agora, basta executar o comando abaixo:

    kubeadm join 192.168.0.19:6443 --token fika1c.4a6rl1ay5yvtkdui \
    --discovery-token-ca-cert-hash sha256:fd6b041e96bd0ec088a8c6e5fff768556e858dcdcced584399dbf5b8dbb51d8e


Como explicado anteriormente, para que os pods se comuniquem com a internet é necessário criar um POD que faça a ligação entre as placas de rede do serviço do Kubernetes com a máquina host. Nesse caso, o comando abaixo irá instalar o POD do Projeto Calico:

    kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
    kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml

Para conferir se todos os PODs do sistema estão em pleno funcionamento, o seguinte comando pode ser dado:

    kubectl get pods --all-namespaces

Se todos estiverem com o status "running", então quer dizer que a instalação foi realizada com sucesso.

## INSTALAÇÃO DO METRICS SERVER

Para instalar o Metrics Server é necessário aplicar o seguinte POD:

    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

Após isso, o pode irá ficar sempre com o status "pending". Para resolver isso, irá precisar editar as configurações do POD:

    kubectl edit deployment -n kube-system metrics-server

Abaixo de:

    --metric-resolution=15s

Adicione:

    command:
    - /metrics-server
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP

Após essa alteração execute o seguinte comando para desabilitar o taint:

    kubectl taint node tcc node-role.kubernetes.io/master:NoSchedule-

## CRIAÇÃO DO AMBIENTE DO PROJETO

Dentro da máquina virtual, deve ser realizado um clone do repositório do GitLab:

    git clone http://192.168.0.19:8484/root/tcc.git

Ele irá pedir as credenciais de acesso. Nesse caso, usaremos o do root.

Após isso, acessaremos a pasta criada:

    cd tcc/

Iniciaremos a criação do deployment:

    kubectl create -f web.yaml

Depois o service:

    kubectl create -f web-service.yaml

E por fim o HPA (Horizontal Pod Autoscaler):

    kubectl create -f web-hpa.yaml

Feito isso, todos os PODs, Deployment e Service serão inciados e devem estar com o status "running"

## EXECUÇÃO DOS TESTES

Para fazer o teste de escalonamento horizontal, será necessário inicialmente a instalação do serviço "siege":

    apt install siege -y

Para execução de requisição em massa, segue um exemplo abaixo:

    siege -q -c 5 -t 2m http://192.168.0.19:PORT

Onde o "PORT" no comando acima se refere a porta que o serviço web irá gerar.

## ACOMPANHAR OS TESTES

Para acompanhar os testes, se faz necessário dois acompanhamentos.

Para todos os recursos:

    watch kubectl get all

Para o POD do serviço web:

    watch kubectl get pods

Para o HPA:

    watch kubectl get hpa

Para os eventos do Kubernetes dos Pods:

    kubectl get events | grep pod

Para os eventos do Kubernetes do HPA:

    kubectl get events | grep horizontalpodautoscaler
