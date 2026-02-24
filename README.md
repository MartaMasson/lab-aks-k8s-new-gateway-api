# Installando o Gateway API em um cluster de AKS já existente coexistindo com NGINX ingress clássico  

Como foi anunciado, o NGINX como ingress clássico vai se [aposentar](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/).

Assim, se você é um usuário do NGINX, precisará migrar para o Gateway API 

Este repositório demonstra como é possível ter múltiplos ingress controllers funcionando em paralelo no mesmo cluster AKS, incluindo o NGINX clássico (tradicional) e novas implementações baseadas na especificação [Kubernetes Gatewat API](https://gateway-api.sigs.k8s.io/). Assim, nos casos em que vai haver impacto nas aplicações clientes que acessam os workloads do cluster (vai haver mudança de endereço Ip, por exemplo), é possível fazer a transição gradual, programada, com fácil reversão em caso de necessidade. 

## Objetivo do Lab

O objetivo deste laboratório é permitir a adequação  ao novo ingress baseado na especificação Kubernetes Gateway API, em um cluster já existente, demonstrando que é possível coexistir diferentes ingress no mesmo cluster AKS sem afetar o funcionamento dos workloads atuais.

Esta abordagem oferece:
- **Migração sem downtime** - Desvio gradual do tráfego entre ingress controllers
- **Teste em produção** - Validação de novas funcionalidades sem impacto total (recomendo executar este lab pelo menos uma vez em um ambiente de teste)
- **Rollback seguro** - Possibilidade de reverter rapidamente se necessário
- **Flexibilidade de escolha** - Diferentes opções de Gateway API para atender necessidades específicas

O lab está estruturado em quatro partes principais:

1. **NGINX Ingress (Clássico)** - Cluster AKS com NGINX como ingress controller gerenciado para simular o ambiente que muita gente tem hoje.
2. **Application Gateway for Containers (AGC)** - Solução Azure nativa para cenários atualmente sem camadas adicionais de proteção (Azure Load Labancer camada 4 com IP público)
3. **Istio Gateway** - Alternativa com capacidades de Service Mesh
4. **Cilium Gateway** - Alternativa com eBPF-based networking

Depois de finalizar a configuração do Istio e Cilium, vai notar que são muito similares. A parte onde há mais mudanças é na parte do GatewayClass, onde cada implementador tem instruções específicas para a instalação do seu gateway. Então, outras implementações (pois há mais do que as mostradas neste lab) vão ficar bem mais fáceis de avaliar e, finalmente, escolher àquela que mais atende aos requisitos da sua solução. [Implementers](https://gateway-api.sigs.k8s.io/implementations/)

## Arquitetura da Solução

### NGINX Tradicional (Ingress Clássico)
- Utiliza a especificação tradicional de Ingress do Kubernetes
- Gerenciado automaticamente pela equipe do AKS através do add-on "App Routing"
- Classe de Ingress: `webapprouting.kubernetes.azure.com`
- Expõe um Load Balancer público com IP externo

### Gateway API - Nova Especificação
Os demais ingress controllers utilizam a nova especificação [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/), que oferece:
- **Maior expressividade** para configurações de roteamento
- **Separação de responsabilidades** entre operadores de infraestrutura e desenvolvedores
- **Padronização** entre diferentes provedores de ingress
- **Recursos mais granulares** (Gateway, HTTPRoute, etc.)

## Estrutura do Laboratório

### Parte 1 - Cluster AKS com NGINX Ingress Clássico

A documentação de base utilizada para criação de um cluster com ingress neginx gerenciado (para facilitar nossa vida aqui ;)) é [essa aqui](https://learn.microsoft.com/en-us/azure/aks/app-routing). A gente só acrescenta o parâmetro --enable-app-routing na criação do cluster e a instalação acontece. Depois, basta criar os manifestos de ingress resources. No lab abaixo, após criar o cluster, vamos fazer o deployment de uma aplicação simples para teste, que estará exposta através do nginx. Esta instalação simula o ambiente que você deve ter aí atualmente.

```powershell

#Variáveis que você pode ajustar
SUBSCRIPTION_ID='id da sua subscrição'
AKS_NAME='aks-new-ingress'
RESOURCE_GROUP='rg-new-ingress'
LOCATION='westus2'
VM_SIZE='Standard_D2s_v6' 

az login
az account set --subscription $SUBSCRIPTION_ID

# É preciso registrar os seguintes resource providers do Azure.
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.NetworkFunction
az provider register --namespace Microsoft.ServiceNetworking

# Criação de um grupo de recursos
az group create --name $RESOURCE_GROUP --location $LOCATION

# Criação do cluster com App Routing add-on habilitado
az aks create --resource-group $RESOURCE_GROUP --name $AKS_NAME --location $LOCATION \
  --node-vm-size $VM_SIZE --enable-app-routing --generate-ssh-keys

#Conectando-se ao AKS
az aks get-credentials --resource-group r$RESOURCE_GROUP --name $AKS_NAME --overwrite-existing

# Criação de um namespace para instalação de uma aplicação de teste que será acessada pelo ingress nginx
kubectl create namespace aks-store

kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/sample-manifests/docs/app-routing/aks-store-deployments-and-services.yaml -n aks-store

# Criação do ingress resource 
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-front
  namespace: aks-store
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - http:
      paths:
      - backend:
          service:
            name: store-front
            port:
              number: 80
        path: /
        pathType: Prefix
EOF

#Capturando o ip exposto para a chamada da aplicação via ingress neginx. 
nginxfqdn=$(kubectl get service -n app-routing-system nginx -o jsonpath="{.status.loadBalancer.ingress[0].ip}")

#Testando a aplicação via nginx ingress
curl -v http://$nginxfqdn/

```

Neste momento, o cluster está instalado com o nginx ingress, servindo a aplicação aks-store.

**Características:**
- NGINX ingress controller pré-instalado e gerenciado pelo AKS
- Configuração através de recursos `Ingress` tradicionais
- Load Balancer público com IP externo disponível imediatamente
- Namespace: `app-routing-system`

### Parte 2 - Application Gateway for Containers (AGC)

Application Gateway for Containers é a implementação da Microsoft para Gateway API no Azure, ideal para cenários onde não há outras camadas de proteção na frente do cluster. Oferece:
- **Integração nativa com Azure** - Gerenciado pelo ALB Controller
- **Proteção sem exposição de IPs públicos** - Tráfego roteado através da infraestrutura Azure
- **Web Application Firewall (WAF) integrado** - Proteção contra ameaças web comuns
- **Escalabilidade automática** - Baseada na demanda
- **SSL/TLS termination** - Certificados gerenciados
- **Classe de Gateway**: `azure-alb-external`
 
É possível trabalhar com Aplication Gateway for Containers de duas formas: 
- Gerenciado, onde todo o ciclo de vida do AGC (provisionamento, configuração dos serviços, aprovisionamento, etc)  é feito através do AKS (managed).
- Bring your own deployment, onde o provisionamento do AGC é feito previamente pelo usuário e se configura o AKS para trabalhar com o AGC. O lab utiliza a versão gerenciada. Documentação de referência  [aqui](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-deploy-application-gateway-for-containers-alb-controller?tabs=install-helm-windows).
 
No nosso laboratório aqui, vamos usar o Application Gateway gerenciado, ou seja, este serviço será provisionado, configurado e mantido pelo AKS. Não devemos tocá-lo. 

**Pre-requisitos para Application Gateway for Containers:**
- Necessária a adicionar a extensão alb
- Workload identity habilitada. Esta funcionalidade permite trabalhar o nível de permissionamento com maior granularidade, além do nível do cluster em si, ou seja, por workload (deployment), que roda dentro do cluster. Para isso será criada uma identidade gerenciada no azure, federada com o AKS. A identidade gerenciada será configurada para ter o permissionamento necessário para provisionamento e gerenciamento do AGC, acesso à configuração de rede, ect pelo controller. 

**Configuração:**

```powershell

#Reset variables
IDENTITY_RESOURCE_NAME='azure-alb-identity'

# Install Azure CLI extensions.
az extension add --name alb

#Updating the AKS cluster to enable OIDC issuer and workload identity. Those features are need to work with ALB controller
az aks update -g $RESOURCE_GROUP -n $AKS_NAME --enable-oidc-issuer --enable-workload-identity --no-wait

#Get resource group of the AKS managed cluster nodepools where AGC will be provisioned.
mcResourceGroup=$(az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query "nodeResourceGroup" -o tsv)
mcResourceGroupId=$(az group show --name $mcResourceGroup --query id -otsv)

#A managed identity is created to Alb controller, federated with AKS (workload identity) needed to provision/configure/delete AGC
echo "AKS managed cluster resource group: $mcResourceGroup"
echo "AKS managed cluster resource group ID: $mcResourceGroupId"
echo "identity name: $IDENTITY_RESOURCE_NAME"
echo "resource group: $RESOURCE_GROUP"

#Create User Assigned Managed Identity for the ALB controller
echo "Creating identity $IDENTITY_RESOURCE_NAME in resource group $RESOURCE_GROUP"
az identity create --resource-group $RESOURCE_GROUP --name $IDENTITY_RESOURCE_NAME
principalId="$(az identity show -g $RESOURCE_GROUP -n $IDENTITY_RESOURCE_NAME --query principalId -otsv)"

echo "Managed Identity principal ID: $principalId"
echo "AKS managed cluster resource group ID: $mcResourceGroupId"

# Contributor role because Microsoft.ServiceNetworking/trafficControllers/write requires it
echo "Apply Contributor role to the AKS managed cluster resource group for the newly provisioned identity"
az role assignment create --assignee-object-id $principalId --assignee-principal-type ServicePrincipal --scope $mcResourceGroupId --role "Contributor" 

#Enable federation between the AKS OIDC issuer and the User Assigned Managed Identity
echo "Set up federation with AKS OIDC issuer"
AKS_OIDC_ISSUER="$(az aks show -n "$AKS_NAME" -g "$RESOURCE_GROUP" --query "oidcIssuerProfile.issuerUrl" -o tsv)"
az identity federated-credential create --name "azure-alb-identity" --identity-name "$IDENTITY_RESOURCE_NAME" --resource-group $RESOURCE_GROUP --issuer "$AKS_OIDC_ISSUER" --subject "system:serviceaccount:azure-alb-system:alb-controller-sa"

echo "AKS OIDC Issuer URL: $AKS_OIDC_ISSUER"

#HELM_NAMESPACE=$CLUSTER_NAME
HELM_NAMESPACE='azure-alb-helm'
CONTROLLER_NAMESPACE='azure-alb-system'
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME
kubectl create namespace $HELM_NAMESPACE
#kubectl create namespace $CONTROLLER_NAMESPACE

#Installing the ALB controller and the gatewayclass via helm 
echo $HELM_NAMESPACE
echo $CONTROLLER_NAMESPACE
echo $RESOURCE_GROUP
echo $IDENTITY_RESOURCE_NAME
helm install alb-controller oci://mcr.microsoft.com/application-lb/charts/alb-controller --namespace $HELM_NAMESPACE --version 1.8.12 --set albController.namespace=$CONTROLLER_NAMESPACE --set albController.podIdentity.clientID=$(az identity show -g $RESOURCE_GROUP -n $IDENTITY_RESOURCE_NAME --query clientId -o tsv)

#Verify the ALB Controller pods are ready. They are responsibile by propagating the Application Gateway for Containers configuration based on the Kubernetes resources.
kubectl get pods -n $CONTROLLER_NAMESPACE

#Verify GatewayClass azure-alb-external is installed on your cluster
kubectl get gatewayclass azure-alb-external -o yaml

#https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-create-application-gateway-for-containers-managed-by-alb-controller?tabs=new-subnet-aks-vnet
#Create Application Gateway for Containers managed by ALB Controller
MC_RESOURCE_GROUP=$(az aks show --name $AKS_NAME --resource-group $RESOURCE_GROUP --query "nodeResourceGroup" -o tsv)
CLUSTER_SUBNET_ID=$(az vmss list --resource-group $MC_RESOURCE_GROUP --query '[0].virtualMachineProfile.networkProfile.networkInterfaceConfigurations[0].ipConfigurations[0].subnet.id' -o tsv)
read -d '' VNET_NAME VNET_RESOURCE_GROUP VNET_ID <<< $(az network vnet show --ids $CLUSTER_SUBNET_ID --query '[name, resourceGroup, id]' -o tsv)

echo "VNET_NAME: $VNET_NAME"
echo "VNET_RESOURCE_GROUP: $VNET_RESOURCE_GROUP"
echo "VNET_ID: $VNET_ID"
echo "MC_RESOURCE_GROUP: $MC_RESOURCE_GROUP"
echo "CLUSTER_SUBNET_ID: $CLUSTER_SUBNET_ID"

#SUBNET_ADDRESS_PREFIX='<network address and prefix for an address space under the vnet that has at least 250 available addresses (/24 or larger subnet)>'
SUBNET_ADDRESS_PREFIX='10.238.1.0/24'
ALB_SUBNET_NAME='subnet-alb' # subnet name can be any non-reserved subnet name (i.e. GatewaySubnet, AzureFirewallSubnet, AzureBastionSubnet would all be invalid)
az network vnet subnet create --resource-group $MC_RESOURCE_GROUP --vnet-name $VNET_NAME --name $ALB_SUBNET_NAME --address-prefixes $SUBNET_ADDRESS_PREFIX --delegations 'Microsoft.ServiceNetworking/trafficControllers'
ALB_SUBNET_ID=$(az network vnet subnet show --name $ALB_SUBNET_NAME --resource-group $MC_RESOURCE_GROUP --vnet-name $VNET_NAME --query '[id]' --output tsv)

echo "ALB_SUBNET_ID: $ALB_SUBNET_ID"
echo "HELM_NAMESPACE: $HELM_NAMESPACE"
echo "CONTROLLER_NAMESPACE: $CONTROLLER_NAMESPACE"
echo "SUBNET_ADDRESS_PREFIX: $SUBNET_ADDRESS_PREFIX"
echo "ALB_SUBNET_NAME: $ALB_SUBNET_NAME"

#Create ApplicationLoadBalancer resource in Kubernetes
kubectl apply -f - <<EOF
apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: alb-loadbalancer-agc
  namespace: $CONTROLLER_NAMESPACE
spec:
  associations:
  - $ALB_SUBNET_ID
EOF

#Additional required permissions
AGC_NAME=$(az network alb list -g "$MC_RESOURCE_GROUP" --query "[0].name" -o tsv)
echo "AGC Name: $AGC_NAME"
AGC_ID=$(az network alb show -g "$MC_RESOURCE_GROUP" -n "$AGC_NAME" --query id -o tsv)
echo "AGC Resource ID: $AGC_ID"
# Delegate AppGw for Containers Configuration Manager role to AKS Managed Cluster RG
az role assignment create --assignee-object-id $principalId --assignee-principal-type ServicePrincipal --scope $mcResourceGroupId --role "AppGw for Containers Configuration Manager"
# Delegate Network Contributor permission for join to association subnet
az role assignment create --assignee-object-id $principalId --assignee-principal-type ServicePrincipal --scope $ALB_SUBNET_ID --role "Network Contributor" 

#verify AGC creation
kubectl get applicationloadbalancer alb-loadbalancer-agc -n $CONTROLLER_NAMESPACE -o yaml 

#https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-ssl-offloading-gateway-api?tabs=alb-managed
#SSL offloading with Application Gateway for Containers - Gateway API
```
Neste ponto, temos o gatewayClass provisionado no AKS. Note  que há um AGC provisionado no grupo de recursos provisionados pelo AKS (onde ficam os nodepools).
Deste ponto, agora os gateways e http route podem ser provisionados. 
Vamos provisionar uma aplicação para que possamos acessar pelo ingress aGC.


```powershell

#Application deployment
kubectl apply -f https://raw.githubusercontent.com/MicrosoftDocs/azure-docs/refs/heads/main/articles/application-gateway/for-containers/examples/https-scenario/ssl-termination/deployment.yaml

```
Depois de realizar o deployment da aplicação, podemos então configurar o Gateway e as HTTPRoutes de aplicações que irão receber o tráfego através do AGC: 

```powershell
#Creating Gateway
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-01
  namespace: test-infra
  annotations:
    #alb.networking.azure.io/alb-namespace: alb-test-infra
    #alb.networking.azure.io/alb-name: alb-test
    alb.networking.azure.io/alb-namespace: $CONTROLLER_NAMESPACE   # ajuste conforme seu ambiente
    alb.networking.azure.io/alb-name: alb-loadbalancer-agc  # ajuste para o nome do seu AGC gerenciado
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: https-listener
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: Same
    tls:
      mode: Terminate
      certificateRefs:
      - kind : Secret
        group: ""
        name: listener-tls-secret
EOF

#Checking Gateway creation
kubectl get gateway gateway-01 -n test-infra -o yaml

#Creating https-route
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-route
  namespace: test-infra
spec:
  parentRefs:
  - name: gateway-01
  rules:
  - backendRefs:
    - name: echo
      port: 80
EOF

#Checking httprouting creation
kubectl get httproute https-route -n test-infra -o yaml

#Getting FQDN of the Gateway
fqdn=$(kubectl get gateway gateway-01 -n test-infra -o jsonpath='{.status.addresses[0].value}')

#Testing ingress
curl --insecure https://$fqdn/
```

Neste ponto, você deveria conseguir acessar a aplicação em execução no cluster através Application Gateway for Containers.

E aqui, alguns pontos de destaque:
- **Sem impacto na aplicação que é acessada pelo nginx** - Você continua acessando normalmente as aplicações que estavam rodando no cluster anteriormente à instalação do AGC.
- **Proteção sem exposição de IPs públicos** - usando Application Gateway for Containers (AGC) corretamente, não existe um IP público externo exposto no cluster (AKS) via Service type=LoadBalancer ou NodePort. O único ponto de entrada público passa a ser o Application Gateway for Containers, que é um recurso Azure L7 fora do data plane do AKS. Portanto, o cluster pode ficar “fechado” do ponto de vista de entrada pública direta, atendendo ao requisito de single entry point via AGC. Isso está alinhado com o design oficial do AGC: o gateway vive fora do cluster, e o AKS não expõe IP público próprio quando você não cria Service LoadBalancer ou NodePort. [learn.microsoft.com].
 
#### Por que o Application Gateway for Containers não expõe IP público?

Uma diferença importante entre o **Application Gateway for Containers (AGC)** e os outros gateways (NGINX, Istio, Cilium) é a forma como o tráfego é exposto externamente.

###### Arquitetura Tradicional dos Outros Gateways

Os gateways tradicionais (NGINX, Istio, Cilium) criam um **Service do tipo LoadBalancer** que:
- Provisiona automaticamente um Azure Load Balancer público
- Obtém um IP público dedicado
- Expõe o tráfego diretamente através deste IP público*1
- O Load Balancer roteia o tráfego diretamente para os pods do ingress controller

*1 Há como resolver isso, obviamente, através do provisonamento de um LB interno via AKS mesmo. 

###### Arquitetura do Application Gateway for Containers

O AGC funciona de forma diferente devido à sua **arquitetura nativa do Azure**:

1. **Recurso Azure Nativo**: O AGC é um recurso Azure de primeira classe, não apenas pods rodando no cluster
2. **Delegação de Subnet**: Utiliza uma subnet delegada (`Microsoft.ServiceNetworking/trafficControllers`) 
3. **Traffic Controller**: O ALB Controller cria um "Traffic Controller" que gerencia o roteamento
4. **Integração com VNET**: O tráfego é roteado através da infraestrutura de rede do Azure, não através de um Load Balancer tradicional
5. **Endereço FQDN**: Em vez de um IP público, fornece um FQDN (Fully Qualified Domain Name)

###### Vantagens desta Abordagem

- **Integração nativa**: Melhor integração com outros serviços Azure
- **Escalabilidade**: Escala automaticamente sem limitações de Load Balancer
- **Segurança**: Tráfego roteado através da infraestrutura segura do Azure
- **Gerenciamento**: Configuração e monitoramento através das ferramentas nativas do Azure
- **Custo**: Otimização de custos através do compartilhamento de infraestrutura. Um único AGC pode servir outros clusters. Neste caso, veja a abordagem Bring your AGC. 

### Parte 3 - Istio Gateway

Istio implementa Gateway API fornecendo uma **alternativa robusta com capacidades de Service Mesh**, ideal para clientes que preferem:
- **Service Mesh capabilities** - Observabilidade, segurança e controle de tráfego avançado
- **Roteamento avançado** - Baseado em headers, pesos, circuit breakers, etc.
- **mTLS automático** - Comunicação segura entre serviços
- **Telemetria completa** - Métricas, logs e traces distribuídos
- **Classe de Gateway**: `istio`


Para instalar o Istio com suporte ao Gateway API, basta seguir as intruções deste [link](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/). Atenção ao manifesto do gateway, que precisa de um ajuste no annotation específico para o Azure, no serviço de loadbalancer do istio-ingressgateway conforme abaixo. Sem essa annotation, o health check do load balancer falha e não vai funcionar. [Documentação de referência sobre o annotation para LB do Azure](https://istio.io/latest/docs/setup/platform-setup/azure/)

```powershell
  annotations:
    service.beta.kubernetes.io/port_80_health-probe_protocol: tcp
```

Após criar o gateway class, os demais manifestos são bem semelhates com relação ao gateway (apontando para o seu respectivo gatewayclass) e httpRoute.

Abaxio, o manifesto do gateway já com a notação do azure incluida:

```powershell
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-02
  namespace: istio-ingress
spec:
  infrastructure:
    annotations:
      service.beta.kubernetes.io/port_80_health-probe_protocol: tcp
  gatewayClassName: istio
  listeners:
  - name: default
    hostname: "*.example.com"
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http
  namespace: default
spec:
  parentRefs:
  - name: gateway-02
    namespace: istio-ingress
  hostnames: ["httpbin.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    backendRefs:
    - name: httpbin
      port: 8000
EOF

#Obtendo o ip exposto para o ingress do istio
kubectl wait -n istio-ingress --for=condition=programmed gateways.gateway.networking.k8s.io gateway-02
export INGRESS_HOST=$(kubectl get gateways.gateway.networking.k8s.io gateway-02 -n istio-ingress -ojsonpath='{.status.addresses[0].value}')

#Testando a chamada da aplicação através do Ip exposto para o Istio
curl -v \
  -H "Host: httpbin.example.com" \
  http://$INGRESS_HOST/get

```

Neste ponto, você deveria conseguir acessar as aplicações que estão em execução de acordo com o ingress ao qual elas foram expostas: NGINX Classico, Application Gateway for Containers e istio.

### Parte 4 - Cilium Gateway

Cilium oferece implementação Gateway API como alternativa para clientes que preferem redes baseadas em eBPF, fornecendo:
- **eBPF-based networking** - Performance otimizada no kernel, removendo a necessidade de existir um sidecar em cada pod.
- **Security policies avançadas** - NetworkPolicies granulares
- **Observabilidade nativa** - Visibilidade completa do tráfego L3/L4/L7  
- **Multicluster networking** - Conectividade entre clusters
- **Classe de Gateway**: `cilium`

Saiba mais sobre redes baseadas em: [eBPF](https://gateway-api.sigs.k8s.io/implementations/#cilium)

A documentação utilizada como base para configurar o Cilium API Gateway controller é [essa aqui](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/#gs-gateway-api), porém ela considera que o cluster é on premises, então eu adaptei para trabalhar com AKS.

```powershell
# Installing the cilium components using Helm repository
helm repo add cilium https://helm.cilium.io/
```

Não é preciso setar o API_SERVER_IP, API_SERVER_PORT, etc conforme indicado na documentação,  porque nosso lab trata-se de um cluster gerenciado e estas variáveis já são configuradas automaticamente ao se conectar no cluster. Entao, removi o desnecessário. Basta seguir os comandos abaixo: 

```powershell
helm install cilium cilium/cilium --version 1.18.6 \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set gatewayAPI.enabled=true

kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium    
```

Neste ponto, o Cilium está instalado com suporte ao Gateway API. Agora vamos criar o gateway e as rotas da mesma forma que fizemos para o Istio.

```powershell
kubectl create namespace cilium-ingress

kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-03
  namespace: cilium-ingress
spec:
  infrastructure:
  gatewayClassName: cilium
  listeners:
  - name: default
    hostname: "*.examplecilium.com"
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpcilium
  namespace: default
spec:
  parentRefs:
  - name: gateway-03
    namespace: cilium-ingress
  hostnames: ["my-nginx.examplecilium.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    backendRefs:
    - name: my-nginx
      port: 8000
EOF
```
E também fazer o deployment de uma aplicação para ser acessada via ingress do cilium, como fizemos também os com demais Gateways APIs.

```powershell

kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-nginx
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    app: my-nginx
    service: my-nginx
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 8080
  selector:
    app: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-nginx
      version: v1
  template:
    metadata:
      labels:
        app: my-nginx
        version: v1
    spec:
      serviceAccountName: my-nginx
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
          hostPort: 8080        
EOF

```
Vamos agora recuperar o IP exposto para receber o tráfego via Gateway API do Cilium e fazer uma chamada de fora do cluster para valiar o acesso ao workload via Cilium.

```powershell

kubectl wait -n cilium-ingress --for=condition=programmed gateways.gateway.networking.k8s.io gateway-03
export INGRESS_CILIUM_HOST=$(kubectl get gateways.gateway.networking.k8s.io gateway-03 -n cilium-ingress -ojsonpath='{.status.addresses[0].value}')

kubectl get gateway gateway-03 -n cilium-ingress -o jsonpath='{.status.addresses[0].value}'

#Teste final de fora do cluster
curl -v \
  -H "Host: examplecilium.example.com" \
  http://$INGRESS_CILIUM_HOST/get
```
Neste ponto, você deveria conseguir acessar as aplicações que estão em execução de acordo com o ingress ao qual elas foram expostas: NGINX Classico, Application Gateway for Containers,  istio e Cilium. Tudo rodando no mesmo cluster. 

CQD ;) 

(Como queríamos demonstrar!)

### Add-on para Gateway API no AKS
Futuramente a Microsoft irá disponibilizar, da mesma forma que existe hoje o addon para NGINX gerenciado, um addon para o istio. A versão preview já está disponível e você pode conferir [aqui](https://blog.aks.azure.com/2025/11/13/ingress-nginx-update)

### Referências

- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [AKS App Routing](https://learn.microsoft.com/azure/aks/app-routing)
- [Application Gateway for Containers](https://learn.microsoft.com/azure/application-gateway/for-containers/)
- [Istio Gateway API](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)
- [Cilium Gateway API](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/)
