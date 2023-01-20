![keda-ado](https://user-images.githubusercontent.com/13451867/213791002-fc46e5fa-3a79-4c5d-92a6-f55814f3bf90.png)

# Utilizando KEDA para escalar agents no Azure Devops no Kubernetes

## English Version
https://github.com/seilorjunior/keda-agents-azure-devops/blob/master/keda-azure-devops-en-us.md

## Introdução

Por padrão temos no Azure Devops a opção de escalar os agents por Azure Virtual machine scale sets [vm-scale-set](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops), porém, temos a opção de utilizar o KEDA para escalar os agents.

Já temos um artigo no blog da Microsoft sobre o [KEDA](https://techcommunity.microsoft.com/t5/desenvolvedores-br/como-escalar-aplica%C3%A7%C3%B5es-no-aks-com-keda/ba-p/3562937), porém, mais a ideia desse artigo é mostrar como utilizar o KEDA para escalar os agents do Azure Devops no Kubernetes.

O código fonte desse artigo está disponível no [github](https://github.com/seilorjunior/keda-agents-azure-devops)

## Pré-requisitos

- Ter um cluster Kubernetes [criando um cluster Kubernetes](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli)
- Ter um container registry [criando um container registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli)
- Integrar o Kubernetes com o container registry [integrando o Kubernetes com o container registry](https://learn.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli)
- Habilitar o KEDA no Kubernetes [habilitando o KEDA no Kubernetes](https://learn.microsoft.com/en-us/azure/aks/keda-deploy-add-on-arm)
- Ter uma organização no Azure Devops [criando uma organização no Azure Devops](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization?view=azure-devops)
- Ter o docker instalado [instalando o docker](https://docs.docker.com/desktop/install/windows-install/)
- Ter o kubeclt instalado [instalando o kubeclt](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- Ter o Azure Cli instalado [instalando o Azure Cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest)

## Configurando o Azure Devops

### Criando um agent pool

Após a criação da organização no Azure Devops, vamos criar um agent pool, para isso, vamos em Organization settings -> Agent pools -> Add pool.

No Pool type selecione Self-hosted e no name digite o nome do pool, no meu caso keda-pool.

No Pipeline permissions deixe marcado Grant access permissions to all pipelines e clique no botão Create.

[create-agent-pool](https://user-images.githubusercontent.com/13451867/212674541-f7a91796-a3e4-494b-a55a-46e1aae2a462.png)

Após a criação do agent pool, será necessário pegar qual é o id do mesmo.

Temos algumas maneiras que como pegar o id do agent pool, uma delas é acessando a url do agent pool, no meu caso https://dev.azure.com/xxxx-sua-organizationn/_apis/distributedtask/pools?poolname=keda-pool, essa chamada vai retornar um json com o id do agent pool.

Abra o json em qualquer ferramenta como o http://jsonviewer.stack.hu/ e procure o id, no meu caso o id é 18 como mostra a imagem abaixo.

![agent-pool-id](https://user-images.githubusercontent.com/13451867/212674648-7f7168e5-1d6d-4f5c-890a-8ef5861857ac.png)

### Criando um PAT no Azure Devops

Vamos precisar de um token do Azure Devops para que o agent consiga se registrar no Azure Devops.

Siga os passos para criar o PAT [criando um PAT no Azure Devops](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows)

Após a criação do PAT, salve o token em algum lugar, pois vamos precisar dele mais a frente.

## Criando uma imagem docker para o agent e publicando no container registry

### Criando um Dockerfile

Para criar uma imagem, siga os passos abaixo.

Crie um arquivo Dockerfile com o conteúdo abaixo.

```dockerfile
FROM ubuntu:20.04
RUN DEBIAN_FRONTEND=noninteractive apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get upgrade -y

RUN DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
    apt-transport-https \
    apt-utils \
    ca-certificates \
    curl \
    git \
    iputils-ping \
    jq \
    lsb-release \
    software-properties-common

RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

# Can be 'linux-x64', 'linux-arm64', 'linux-arm', 'rhel.6-x64'.
ENV TARGETARCH=linux-x64

WORKDIR /azp

COPY ./start.sh .
RUN chmod +x start.sh

ENTRYPOINT [ "./start.sh" ]

```

Na mesma pasta que foi criado o Dockerfile, crie um arquivo chamado start.sh com o conteúdo abaixo.

```bash
#!/bin/bash
set -e

if [ -z "$AZP_URL" ]; then
  echo 1>&2 "error: missing AZP_URL environment variable"
  exit 1
fi

if [ -z "$AZP_TOKEN_FILE" ]; then
  if [ -z "$AZP_TOKEN" ]; then
    echo 1>&2 "error: missing AZP_TOKEN environment variable"
    exit 1
  fi

  AZP_TOKEN_FILE=/azp/.token
  echo -n $AZP_TOKEN > "$AZP_TOKEN_FILE"
fi

unset AZP_TOKEN

if [ -n "$AZP_WORK" ]; then
  mkdir -p "$AZP_WORK"
fi

export AGENT_ALLOW_RUNASROOT="1"

cleanup() {
  if [ -e config.sh ]; then
    print_header "Cleanup. Removing Azure Pipelines agent..."

    # If the agent has some running jobs, the configuration removal process will fail.
    # So, give it some time to finish the job.
    while true; do
      ./config.sh remove --unattended --auth PAT --token $(cat "$AZP_TOKEN_FILE") && break

      echo "Retrying in 30 seconds..."
      sleep 30
    done
  fi
}

print_header() {
  lightcyan='\033[1;36m'
  nocolor='\033[0m'
  echo -e "${lightcyan}$1${nocolor}"
}

# Let the agent ignore the token env variables
export VSO_AGENT_IGNORE=AZP_TOKEN,AZP_TOKEN_FILE

print_header "1. Determining matching Azure Pipelines agent..."

AZP_AGENT_PACKAGES=$(curl -LsS \
    -u user:$(cat "$AZP_TOKEN_FILE") \
    -H 'Accept:application/json;' \
    "$AZP_URL/_apis/distributedtask/packages/agent?platform=$TARGETARCH&top=1")

AZP_AGENT_PACKAGE_LATEST_URL=$(echo "$AZP_AGENT_PACKAGES" | jq -r '.value[0].downloadUrl')

if [ -z "$AZP_AGENT_PACKAGE_LATEST_URL" -o "$AZP_AGENT_PACKAGE_LATEST_URL" == "null" ]; then
  echo 1>&2 "error: could not determine a matching Azure Pipelines agent"
  echo 1>&2 "check that account '$AZP_URL' is correct and the token is valid for that account"
  exit 1
fi

print_header "2. Downloading and extracting Azure Pipelines agent..."

curl -LsS $AZP_AGENT_PACKAGE_LATEST_URL | tar -xz & wait $!

source ./env.sh

print_header "3. Configuring Azure Pipelines agent..."

./config.sh --unattended \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --url "$AZP_URL" \
  --auth PAT \
  --token $(cat "$AZP_TOKEN_FILE") \
  --pool "${AZP_POOL:-Default}" \
  --work "${AZP_WORK:-_work}" \
  --replace \
  --acceptTeeEula & wait $!

print_header "4. Running Azure Pipelines agent..."

if ! grep -q "template" <<< "$AZP_AGENT_NAME"; then
  echo "Cleanup Traps Enabled"

  trap 'cleanup; exit 0' EXIT
  trap 'cleanup; exit 130' INT
  trap 'cleanup; exit 143' TERM

fi

chmod +x ./run-docker.sh

# To be aware of TERM and INT signals call run.sh
# Running it with the --once flag at the end will shut down the agent after the build is executed
./run-docker.sh "$@" --once & wait $!
```

O seu arquivo start.sh tem que ter o EOL do unix/linux , caso esteja no windows, você pode usar o [dos2unix](https://sourceforge.net/projects/dos2unix/) para converter o arquivo ou utilizar o notepad++ e configurar o EOL para unix/linux.

![eol-conversion](https://user-images.githubusercontent.com/13451867/212674705-9d231f6e-04a9-47ae-930e-33d7b4e08746.png)

### Criando a imagem e publicando no ACR

Abra um terminal na pasta onde está o Dockerfile e o start.sh e execute o comandos abaixo.

```bash
# cria a imagem
docker build -t azure-devops-agent:latest .
# loga no seu ACR
docker login -u <user> -p <password> <seu acr name>
# cria a tag da imagem
docker tag azure-devops-agent:latest <seu acr name>/azure-devops-agent:latest
# envia a imagem para o ACR
docker push <seu acr name>/azure-devops-agent:latest
```

## Criando os arquivos YML para o Kubernetes

### Criando o deployment do Azure DevOps Agent

Pegue o PAT que foi gerado nos passos anteriores e vamos precisar converter o mesmo para base64.

Va no site https://base64.guru/converter e cole o PAT gerado e clique em encode e salve o valor gerado.

Crie um arquivo chamado deployment.yml com o conteúdo abaixo.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azdevops
data:
  AZP_TOKEN: 'seu pat gerado convertido para base64'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azdevops-deployment
  labels:
    app: azdevops-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azdevops-agent
  template:
    metadata:
      labels:
        app: azdevops-agent
    spec:
      containers:
      - name: azdevops-agent
        #nome do seu ACR que foi criado
        image: nomedoseuacr.azurecr.io/azure-devops-agent:latest
        env:
          - name: AZP_URL
            value: "https://dev.azure.com/sua-organizacao" #url da sua organização
          - name: AZP_POOL
            value: "keda-pool"
          - name: AZP_TOKEN
            valueFrom:
              secretKeyRef:
                name: azdevops
                key: AZP_TOKEN
```

### Fazendo o deploy do deployment no Kubernetes

Primeiro vamos instalar Kubernetes CLI para o Azure, fazer o login no cluster e executar o deploy.

```bash
# instalando o Kubernetes CLI
az aks install-cli
# logando no seu cluster
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
# criando um namespace para o seu deployment
kubectl create namespace azdevops
# fazendo o deploy do deployment
kubectl apply -f deployment.yml -n azdevops
# verificando se o pod foi criado
kubectl get pods -n azdevops
# vai ter uma resposta parecido com essa
NAME                                   READY   STATUS    RESTARTS   AGE
azdevops-deployment-86b9c58cbd-nm45l   1/1     Running   0          13s
```

Agora vá no seu Azure Devops no pool que criamos no passo anterior e verifique se o agente foi adicionado e verifique se ele está online.

![pool-without-scale](https://user-images.githubusercontent.com/13451867/212674742-2597ea00-7b4e-4e54-bddb-1006d91c39d6.png)

### Criando um pipeline

Crie um projeto default na sua organização do Azure DevOps. [Clique aqui para saber como criar um projeto](https://docs.microsoft.com/pt-br/azure/devops/organizations/projects/create-project?view=azure-devops&tabs=preview-page)

Após a criação do projeto, vamos criar um pipeline. [Clique aqui para saber como criar um pipeline](https://learn.microsoft.com/en-us/azure/devops/pipelines/create-first-pipeline?view=azure-devops&tabs=net%2Ctfs-2018-2%2Cbrowser)

Quando aparecer a opção do review do seu pipeline, mude o pool para o que criamos, no meu caso foi o keda-pool e clique save and run.

![save-run-pipeline](https://user-images.githubusercontent.com/13451867/212674775-f7af5f5f-965d-4c4a-854b-2ea152873c74.png)

Vá no seu pool e verifique se o agente está rodando a build.

![keda-pool-build](https://user-images.githubusercontent.com/13451867/212674819-794271b2-17ce-48fb-95b4-74c577924966.png)

### Criando o Scale Object do KEDA

Crie um arquivo chamado scale-object.yml com o conteúdo abaixo.

Esse padrão de arquivo do KEDA, vai fazer o scaling por deployment, ao executar seus agentes como deployment, você não tem controle sobre qual pod é killed ao reduzir o scaling.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pipeline-auth
data:
  personalAccessToken: 'seu pat gerado convertido para base64'
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: pipeline-trigger-auth
spec:
  secretTargetRef:
    - parameter: personalAccessToken
      name: pipeline-auth
      key: personalAccessToken
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-pipelines-scaledobject
spec:
  scaleTargetRef:
    name: azdevops-deployment
  minReplicaCount: 1
  maxReplicaCount: 5 #número máximo de replicas
  triggers:
  - type: azure-pipelines
    metadata:
      poolID: "18" #seu id do pool
      organizationURLFromEnv: "AZP_URL"
    authenticationRef:
     name: pipeline-trigger-auth
```

Vamos fazer o deploy do arquivo scale-object.yml, rode os comandos abaixo.

```bash
# cria os arquivo necessários para o KEDA
kubectl apply -f scale-object.yml -n azdevops
# verifica se o scaled object foi criado
kubectl get scaledobject -n azdevops
# vai ter uma resposta parecido com essa
NAME                           SCALETARGETKIND      SCALETARGETNAME       MIN   MAX   TRIGGERS          AUTHENTICATION          READY   ACTIVE   FALLBACK   AGE
azure-pipelines-scaledobject   apps/v1.Deployment   azdevops-deployment   1     5     azure-pipelines   pipeline-trigger-auth   True    False    False      73s
```

Após o deploy dos manifestos do KEDA, vamos executar o mesmo pipeline várias vezes para ver o scaling acontecer.

![scaling-per-deployment](https://user-images.githubusercontent.com/13451867/212674849-7bbc2b00-6680-46c9-8b68-eb54ba96121c.png)

Como podemos ver na imagem acima, o agent foi escalado para 4, pois temos 4 pipelines rodando. Caso tenha um pipeline que rode 4 jobs em paralelo o agent também será escalado para 4.

## Conclusão

Como visto com o KEDA temos mais uma opção para escalar os agents de uma maneira simples e eficiente, lembrando que qualquer outra tools necessária para o seu processo de build/deploy tem que ser instalada junto no seu Dockerfile.

O KEDA também suporta mais alguns parâmetros para trigger, como podemos ver abaixo.

```yaml
triggers:
  - type: azure-pipelines
    metadata:
      # Optional: Name of the pool in Azure DevOps
      poolName: "{agentPoolName}"
      # Optional: Learn more in 'How to determine your pool ID'
      poolID: "{agentPoolId}"
      # Optional: Azure DevOps organization URL, can use TriggerAuthentication as well
      organizationURLFromEnv: "AZP_URL"
      # Optional: Azure DevOps Personal Access Token, can use TriggerAuthentication as well
      personalAccessTokenFromEnv: "AZP_TOKEN"
      # Optional: Target queue length
      targetPipelinesQueueLength: "1" # Default 1
      activationTargetPipelinesQueueLength: "5" # Default 0
      # Optional: Parent template to read demands from
      parent: "{parent ADO agent name}"
      # Optional: Demands string to read demands from ScaledObject
      demands: "{demands}"
    authenticationRef:
     name: pipeline-trigger-auth
```

## Referências

1. [Example for ScaledObject](https://keda.sh/docs/2.9/scalers/azure-pipelines/#example-for-scaledobject)
1. [Azure Pipelines scaler](https://keda.sh/docs/2.9/scalers/azure-pipelines/)
1. [Docker Agents in Azure Devops](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#add-tools-and-customize-the-container)
