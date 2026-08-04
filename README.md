# Backstage-Testbed

How to setup a viable backstage personal development system with a database backend and github auth.

---
## Pre-reqs

### Github account

You'll need a github account, naturally. The part we are after is the username, e.g github.com/__bob__

```bash
export GITHUB_USER="bob"
```

### Github Organisation

You can create one here at [https://github.com/settings/organizations](https://github.com/settings/organizations). You might be able to do this without, but i did and it works.

```bash
export GITHUB_ORG="my_github_org"
```
### This repo

Download the repo and cd into it.

```bash
git clone https://github.com/mv-2112/Backstage-Testbed.git && cd Backstage-Testbed
```

---

## Prepare the files

This repo contains the files you will need modify with your specifics.

| File             | Description                                                              |
| -----------------| ------------------------------------------------------------------------ |
| Dockerfile       | This Dockerfile will instruct how to build the backstage container       |
| pg_values.yaml   | Contains the config for PostgreSQL                                       |
| app-config.yaml  | The application config, requires changes                                 |
| App.tsx          | Modified file with SignInPage routine                                    |
| index.ts         | Modified file with additional featues imported                           |

### Dockerfile

This requires no changes. The sqlite3 section can probably be removed.

### pg_values.yaml

You may wish to allocate more space than 1GB

### app_config.yaml

This requires a number of changes, these will be guided in a later section

### App.tsx

This file is ready to go, you will copy this in place later

### index.tx

This file is ready to go, you will copy this in place later

---

## Install docker, k8s and other software

This has been tested on Ubuntu 26.04 LTS using microk8s, its my preferred option but any alternative that gives you storage for PV's, dns, ingress and cert-manager and metallb should work fine.

### Install microk8s

```bash
sudo snap install microk8s --classic
```

You can install the `kubectl` snap
```bash
sudo snap install kubectl --classic
```

Or alias the microk8s built-in
```bash
alias kubectl='microk8s kubectl'
```

```bash
sudo usermod -a -G microk8s $USER
newgrp microk8s
```

### Install Docker

We will also need docker to build the Backstage container later on. You can use podman if you prefer.

```bash
sudo snap install docker
sudo groupadd docker
sudo chgrp docker /var/run/docker.sock
sudo usermod -a -G docker $USER
newgrp docker
```
### Install Helm

```bash
sudo snap install helm --classic
```
Or alias the microk8s built-in helm
```bash
alias helm='microk8s helm'
```

### Configure microk8s

Enable the required extra features. You should change the metallb network range to suit your network - ideally they should be out of your DHCP range
```bash
microk8s enable hostpath-storage
microk8s enable dns
microk8s enable ingress
microk8s enable cert-manager
```
Enable metallb with an IP range applicable to your machines subnet
```bash
microk8s enable metallb: 10.225.118.20-10.225.118.39
```
### Install node.js

Install NVM
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.6/install.sh | bash
```

Setup your env by running the commands below (or restart your terminal)
```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

Install the latest LTS release of node.js
```bash
nvm install --lts
```
---

## Installing PostgreSQL

### Install krew

__TODO__: review if there is a simpler way as per 
https://krew.sigs.k8s.io/docs/user-guide/setup/install/

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```
This requires changes to your enf file.
```bash
echo "export PATH=\"${KREW_ROOT:-$HOME/.krew}/bin:$PATH\"" >> ~/.bashrc
```
Either restart your shell or run direct
```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

Test the krew install works and add cnpg (Cloud Native PostgreSQL)

```bash
kubectl-krew update
kubectl-krew install cnpg
```

### Install Cloud Native Postgresql

If you are using microk8s, you need to ensure your kubeconfig is in the right place for other tools.

```bash
if [[ ! -d $HOME/.kube ]]; then
  echo "Creating $HOME/.kube directory" 
  mkdir $HOME/.kube
  echo "Creating kubeconfig in $HOME/.kube/config"
  microk8s config > $HOME/.kube/config
elif [[ ! -f $HOME/.kube/config ]]; then
  echo "Creating kubeconfig in $HOME/.kube/config"
  microk8s config > $HOME/.kube/config
fi
```

Now follow from https://github.com/pgEdge/pgedge-helm/blob/main/docs/install.md#step-2-install-chart-dependencies

#### Add the pgEdge Helm repository

```bash
helm repo add pgedge https://pgedge.github.io/charts
helm repo update
```
### Install CloudNativePG operator

```bash
helm install cnpg pgedge/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace
```

### Deploy CloudNativePG instance

Create a pg_values.yaml from https://github.com/pgEdge/pgedge-helm/blob/main/examples/configs/single/values.yaml or use the one in this repo.
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: backstage-database
  namespace: backstage
spec:
  instances: 1
  enableSuperuserAccess: true
  monitoring:
    enablePodMonitor: true
  storage:
    size: 10Gi
```
Run the below to install.
```bash
kubectl apply -f ./pg_values.yaml
```
---

## Building Backstage

More information at https://backstage.io/docs/getting-started/

```bash
npm install -g corepack
npx @backstage/create-app@latest
```

```console
$ npx @backstage/create-app@latest
Need to install the following packages:
@backstage/create-app@0.9.0
Ok to proceed? (y) y
? Enter a name for the app [required] backstage
```


```bash
cd backstage
```

Install modules for github auth and catalog sync (for users).
```bash
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
yarn --cwd packages/backend add @backstage/plugin-catalog-backend-module-github-org
```

Remove the production config
```bash
mv app-config.production.yaml app-config.production.yaml.donotuse
```

Copy App.tsx and index.ts into place

```bash
cp ../App.tsx ./packages/app/src
cp ../index.ts ./packages/backend/src
```

Now lets make changes to the app-config.yaml file, this is the core config.
```bash
yq -i '.app.baseUrl="https://backstage.local"' ./app-config.yaml
yq -i '.backend.cors.origin="https://backstage.local"' ./app-config.yaml
yq -i '.backend.baseUrl="https://backstage.local"' ./app-config.yaml

yq -i '.backend.database = {"client": "pg", "connection": {"host": "${POSTGRES_HOST}", "port": 5432, "user": "${POSTGRES_USER}", "password": "${POSTGRES_PASSWORD}"}}' ./app-config.yaml

yq -i 'with(.integrations.github[0]; . *= {"host": "github.com", "apps": {"appId": "${GITHUB_APP_ID}", "clientId": "${GITHUB_APP_CLIENT_ID}", "clientSecret": "${GITHUB_APP_CLIENT_SECRET}", "privateKey": "${GITHUB_APP_PRIVATE_KEY}"}} | del(.token))' ./app-config.yaml


yq -i '.auth = {"clientIdMetadataDocuments": {"enabled": false}, "environment": "production", "providers": {"github": {"production": {"clientId": "${GITHUB_APP_CLIENT_ID}", "clientSecret": "${GITHUB_APP_CLIENT_SECRET}", "signIn": {"resolvers": [{"resolver": "usernameMatchingUserEntityName"}]}}}}}' ./app-config.yaml

yq -i '.scaffolder.experimentalTemplateEditor = true' ./app-config.yaml


yq -i ' .catalog = {"providers": {"githubOrg": [{"id": "production", "githubUrl": "https://github.com", "orgs": [env(GITHUB_ORG)], "schedule": {"frequency": {"minutes": 10}, "timeout": {"minutes": 5}}}]}}' ./app-config.yaml
  
yq -i ' .permission = {"enabled": true, "options": {"adminUsers": [("user:default/" + env(GITHUB_USER))]}} ' ./app-config.yaml
```


From https://backstage.io/docs/deployment/docker#host-build, lets build the application.

```bash
yarn install --immutable
yarn tsc
yarn build:backend
```



## Build your Docker images and push to your docker account

Copy the Dockerfile into palce
```bash
cp ../Dockerfile .
```

Build and upload the Docker image
```bash
docker build -t $DOCKER_USER/backstage:0.0.15 .
docker push $DOCKER_USER/backstage:0.0.15
```



---

__TODO__

## Create a Github app

```bash
yarn backstage-cli create-github-app blackcatengineering
```



## Might be needed for DB
kubectl exec -it backstage-database-1 -n backstage -- psql -U postgres -d postgres -c "ALTER USER app CREATEDB;"



## Deploy Backstage



## Deploy Backstage to your cluster
kubectl apply -f backstage-prod.yaml
