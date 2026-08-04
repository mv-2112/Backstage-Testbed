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
__TODO__
- alter files
- create github org... or personal?
  


From https://backstage.io/docs/deployment/docker#host-build
```bash
yarn install --immutable
yarn tsc
yarn build:backend
```



Create a Github app

```bash
yarn backstage-cli create-github-app blackcatengineering
```

Create this Dockerfile in the backstage directory
```Dockerfile
FROM node:24-trixie-slim

# Set Python interpreter for `node-gyp` to use
ENV PYTHON=/usr/bin/python3

# Install isolate-vm dependencies, these are needed by the @backstage/plugin-scaffolder-backend.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 g++ build-essential && \
    rm -rf /var/lib/apt/lists/*

# From here on we use the least-privileged `node` user to run the backend.
USER node

# This should create the app dir as `node`.
# If it is instead created as `root` then the `tar` command below will fail: `can't create directory 'packages/': Permission denied`.
# If this occurs, then ensure BuildKit is enabled (`DOCKER_BUILDKIT=1`) so the app dir is correctly created as `node`.
WORKDIR /app

# Copy files needed by Yarn
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./
COPY --chown=node:node backstage.json ./

# This switches many Node.js dependencies to production mode.
ENV NODE_ENV=production

# This disables node snapshot for Node 20 to work with the Scaffolder
ENV NODE_OPTIONS="--no-node-snapshot"

# Copy repo skeleton first, to avoid unnecessary docker cache invalidation.
# The skeleton contains the package.json of each package in the monorepo,
# and along with yarn.lock and the root package.json, that's enough to run yarn install.
COPY --chown=node:node yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"

# This will include the examples, if you don't need these simply remove this line
COPY --chown=node:node examples ./examples

# Then copy the rest of the backend bundle, along with any other files we might want.
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

CMD ["node", "packages/backend", "--config", "app-config.yaml"]
```

## Build your Docker images and push to your docker account

```bash
docker build -t verranm/backstage:0.0.15 .
docker push verranm/backstage:0.0.15
```


## Might be needed for DB
kubectl exec -it backstage-database-1 -n backstage -- psql -U postgres -d postgres -c "ALTER USER app CREATEDB;"




## Deploy Backstage to your cluster
kubectl apply -f backstage-prod.yaml
