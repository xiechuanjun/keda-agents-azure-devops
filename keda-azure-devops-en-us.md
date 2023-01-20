![keda-ado](https://user-images.githubusercontent.com/13451867/213790939-5ea90900-dc99-435e-ac97-6df516665294.png)


# Introduction

The default option to have a scalable self-hosted pool in Azure Devops is use [Azure virtual machine scale sets](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops). However, if you have an ecosystem that is running mostly in containers and Kubernetes, you might also think about using containers instead and leverage KEDA scalable triggers to spin up more agents when the demand increases and lower them when the demand decreases.

Compared to Azure VM scale sets, the Kubernetes instances are much faster to create (given you have available resources in the cluster and don't need to create new nodes).

The purpose for this article is to quickly demonstrate a working solution for this scenario. It is not the purpose to show how to work with KEDA because we have so many other [articles](https://techcommunity.microsoft.com/t5/forums/searchpage/tab/message?advanced=false&allow_punctuation=false&q=keda) in TechCommunity already for this topic.

The source code for this exercise is [here](https://github.com/seilorjunior/keda-agents-azure-devops).

## Pre-requisites

- [Azure Kubernetes cluster](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster?tabs=azure-cli)
- [Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli)
- [Integrated Azure Container Registry with AKS](https://learn.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli)
- [KEDA installed in the AKS cluster](https://learn.microsoft.com/en-us/azure/aks/keda-deploy-add-on-arm). Alternatively, you might also install with [Helm](https://keda.sh/docs/2.9/deploy/#helm)
- [Azure DevOps organization](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization?view=azure-devops)
- [Azure DevOps configured to have at least 5 parallel jobs for self-hosted private projects](https://learn.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs?view=azure-devops&tabs=self-hosted#how-do-i-buy-more-parallel-jobs)
- [Docker](https://docs.docker.com/desktop/install/windows-install/)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest)

## Setting up Azure Devops

### Creating an agent pool

After you have your organization created in Azure Devops, let's create a new agent pool. For that, go to Organization settings -> Agent pools -> Add pool.

In the pool type, select "Self-hosted" and give it a name like "keda-pool". Then, check the box "Grant access permissions to all pipelines" and click "Create".

![create-agent-pool](https://user-images.githubusercontent.com/13451867/212674541-f7a91796-a3e4-494b-a55a-46e1aae2a462.png)

After the agent pool is created, we need to get its **id**.

One way to get the agent pool id is to access this URL: https://dev.azure.com/**your-organization-name**/_apis/distributedtask/pools?poolname=keda-pool. The response contains a JSON with its id (in my case, its "18"):

![agent-pool-id](https://user-images.githubusercontent.com/13451867/212674648-7f7168e5-1d6d-4f5c-890a-8ef5861857ac.png)

### Generating a PAT (Personal Access Token) in Azure Devops

It is necessary to have an Azure Devops token to be able to register our agents in the agent pool.

Follow [these steps](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows) to create a PAT.

Don't forget to save your token because we will need it later.

## Building a Docker image for the agent and publishing it to the container registry

### Creating a Dockerfile

Create a file named "Dockerfile" with the following contents:

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

In the same folder where you created your Dockerfile, add another file named "start.sh" with the following contents:

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

Just be aware that you need to set the EOL (end of line) character for these files to Unix/Linux format LF. You can do that by using a text/code editor like Visual Studio Code or Notepad++ for instance.

![eol-conversion](https://user-images.githubusercontent.com/13451867/212674705-9d231f6e-04a9-47ae-930e-33d7b4e08746.png)

### Building the image and publishing to ACR

Open a terminal window in the folder where you have Dockerfile and start.sh and execute the following commands:

```bash
# Build the image
docker build -t azure-devops-agent:latest .
# Login to ACR
docker login -u <user> -p <password> <your ACR address>
# Tag the image
docker tag azure-devops-agent:latest <your ACR address>/azure-devops-agent:latest
# Send image to ACR
docker push <your ACR address>/azure-devops-agent:latest
```

## Setting up Kubernetes

### Creating Kubernetes YML manifests for the Azure DevOps Agent

Take the PAT generated previously and convert it to base64.

Add a file named "deployment.yml" with the following content:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azdevops
data:
  AZP_TOKEN: '<your base64 PAT>' #replace
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
        image: <your ACR address>/azure-devops-agent:latest  #replace 
        env:
          - name: AZP_URL
            value: "https://dev.azure.com/<your organization name>" #replace
          - name: AZP_POOL
            value: "keda-pool"
          - name: AZP_TOKEN
            valueFrom:
              secretKeyRef:
                name: azdevops
                key: AZP_TOKEN
```

### Deploying to Kubernetes

Let's install the AKS CLI, login to the cluster e run the deploy commands:

```bash
# Installing AKS CLI
az aks install-cli
# Logging in
az aks get-credentials --resource-group <myResourceGroup> --name <myAKSCluster>
# Create a namespace
kubectl create namespace azdevops
# Deploying to AKS
kubectl apply -f deployment.yml -n azdevops
# Check if the pod is up and running
kubectl get pods -n azdevops
# You should see an output similar to:
NAME                                   READY   STATUS    RESTARTS   AGE
azdevops-deployment-86b9c58cbd-nm45l   1/1     Running   0          13s
```

In the Azure DevOps portal, check if your new agent is correctly showing up in the agent pool:

![pool-without-scale](https://user-images.githubusercontent.com/13451867/212674742-2597ea00-7b4e-4e54-bddb-1006d91c39d6.png)

### Creating a pipeline

Add a new project to your Azure DevOps organization: [doc](https://docs.microsoft.com/en-us/azure/devops/organizations/projects/create-project?view=azure-devops&tabs=preview-page)
Then, create a new pipeline: [doc](https://learn.microsoft.com/en-us/azure/devops/pipelines/create-first-pipeline?view=azure-devops&tabs=net%2Ctfs-2018-2%2Cbrowser)

In the end of the pipeline creation wizard, change the pool name do "keda-pool" and save/run it:

![save-run-pipeline](https://user-images.githubusercontent.com/13451867/212674775-f7af5f5f-965d-4c4a-854b-2ea152873c74.png)

Go back to the Agent Pool page and check if your new agent is running the pipeline:

![keda-pool-build](https://user-images.githubusercontent.com/13451867/212674819-794271b2-17ce-48fb-95b4-74c577924966.png)

### Adding KEDA to dynamically provision new agent instances

Great, so far you have a single agent. That means if you try to build multiple pipelines at the same time with it, they will queue up and run sequentially.
Let's leverage KEDA to automatically provision a new instance of the agent for each pipeline run... with the maximum of 5 instances.

Create a file named "scale-object.yml" with the following content:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pipeline-auth
data:
  personalAccessToken: '<your base64 PAT>' #replace
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
  maxReplicaCount: 5 #Maximum number of parallel instances
  triggers:
  - type: azure-pipelines
    metadata:
      poolID: "<Agent Pool Id>" #Replace with your agent pool ID
      organizationURLFromEnv: "AZP_URL"
    authenticationRef:
     name: pipeline-trigger-auth
```

And then deploy the manifest to Kubernetes:

```bash
kubectl apply -f scale-object.yml -n azdevops
# Verify if it is created
kubectl get scaledobject -n azdevops
# You should see something similar to
NAME                           SCALETARGETKIND      SCALETARGETNAME       MIN   MAX   TRIGGERS          AUTHENTICATION          READY   ACTIVE   FALLBACK   AGE
azure-pipelines-scaledobject   apps/v1.Deployment   azdevops-deployment   1     5     azure-pipelines   pipeline-trigger-auth   True    False    False      73s
```

Now, run your new pipeline many times (without waiting for the previous run to complete) and check how the number of agents increase in the agent pool:

![scaling-per-deployment](https://user-images.githubusercontent.com/13451867/212674849-7bbc2b00-6680-46c9-8b68-eb54ba96121c.png)

In this case, I ran the pipeline 4 times and I got 4 agents in Kubernetes to handle the deployments. If these agents do not get a new job within 5 minutes (the default KEDA scaling down timeout), the pods will be killed in Kubernetes until we are down to 1 single agent again.

## Conclusion

Using KEDA is a fast and cost efficient way to scale your Azure DevOps pipelines if you are comfortable working with containers and Kubernetes.
Just be aware this tutorial covered a basic scenario. KEDA provides other options to configure the scaling behavior as you can see below:

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

## References

1. [Example for ScaledObject](https://keda.sh/docs/2.9/scalers/azure-pipelines/#example-for-scaledobject)
1. [Azure Pipelines scaler](https://keda.sh/docs/2.9/scalers/azure-pipelines/)
1. [Docker Agents in Azure Devops](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#add-tools-and-customize-the-container)
