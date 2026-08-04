# Backstage-Testbed
How to setup a viable backstage personal development system




## Install k8s

This has been tested on microk8s, its my preferred option but any alternative that gives you storage for PV's, dns, ingress and cert-manager and metallb will work fine.

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


## Installing the PostgreSQL backend

### Install krew

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

```bash
echo "export PATH=\"${KREW_ROOT:-$HOME/.krew}/bin:$PATH\"" >> ~/.bashrc
```

```bash
kubectl-krew update
kubectl-krew install cnpg
```

### Install Helm

```bash
sudo snap install helm --classic
```
Or alias the microk8s built-in helm
```bash
alias helm='microk8s helm'
```

### Install Cloud Native Postgresql

cd $HOME
mkdir .kube
cd .kube
microk8s config > config

Now follow from https://github.com/pgEdge/pgedge-helm/blob/main/docs/install.md#step-2-install-chart-dependencies

#### Add the pgEdge Helm repository
helm repo add pgedge https://pgedge.github.io/charts
helm repo update

#### Install CloudNativePG operator
helm install cnpg pgedge/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace


### Deploy CloudNativePG instance

Create a values.yaml from https://github.com/pgEdge/pgedge-helm/blob/main/examples/configs/single/values.yaml
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
helm install pgedge pgedge/pgedge   --values ./values.yaml   --wait -n backstage  --create-namespace
```
  
## Install node.js
 
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.6/install.sh | bash
```

## Building Backstage

More information at https://backstage.io/docs/getting-started/

```bash
npm install -g corepack
npx @backstage/create-app@latest
cd backstage
```

From https://backstage.io/docs/deployment/docker#host-build
```bash
yarn install --immutable
yarn tsc
yarn build:backend
```

Install modules for github auth and catalog sync (for users).
```bash
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
yarn --cwd packages/backend add @backstage/plugin-catalog-backend-module-github-org
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
