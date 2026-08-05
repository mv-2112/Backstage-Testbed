# Backstage-Testbed

How to setup a viable backstage personal development system with a database backend and github auth.

This was created and proven on Ubuntu 26.04 LTS. You can use something like Virtualbox to run Ubuntu on Windows if you like. (Also useful on Linux too if you want to leave your main install pristine). Unfortunately Multipass is not an option as Metallb won't work there.

---
## Pre-reqs

### Docker account

You'll need a Docker account, the part we are after is the repository name, e.g hub.docker.com/repositories/__bob__

```bash
export DOCKER_REPO="bob"
```

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
### Install gh 

```bash
sudo snap install gh --classic
```
Use `gh` to authenticate, sample shown below.
```bash
$ gh auth login
? Where do you use GitHub? GitHub.com
? What is your preferred protocol for Git operations on this host? HTTPS
? Authenticate Git with your GitHub credentials? Yes
? How would you like to authenticate GitHub CLI? Login with a web browser

! First copy your one-time code: 414C-B407
Press Enter to open https://github.com/login/device in your browser... 
! Failed opening a web browser at https://github.com/login/device
  exec: "xdg-open,x-www-browser,www-browser,wslview": executable file not found in $PATH
  Please try entering the URL in your browser manually
✓ Authentication complete.
- gh config set -h github.com git_protocol https
✓ Configured git protocol
! Authentication credentials saved in plain text
✓ Logged in as mv-2112
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
microk8s enable metallb 10.225.118.20-10.225.118.39
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

### Create CloudNativePG config

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

### Deploy CloudNativePG instance

```bash
kubectl create ns backspace
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

Copy the Dockerfile into place
```bash
cp ../Dockerfile .
```

Build and upload the Docker image
```bash
docker build -t $DOCKER_REPO/backstage:0.0.15 .
docker push $DOCKER_REPO/backstage:0.0.15
```



---

## Create a Github app

This step requires a browser, so if running from Multipass or Virtualbox without a GUI it may not work. Use the __Manual method__ instead.

### Manual method

In your github Organisation, open its settings (make sure it is your Org and not your account) and expand the __Developer Settings__ section. Click __Github Apps__.

Click __New Github App__

#### Key fields
The key fields to populate are:-

| Field | Value |
|-------|-------|
|GitHub App name| Backstage-yournamehere|
|Homepage URL|https://backstage.local|
|Callback URL|https://backstage.local|
|Webhook| Inactive, clear the tick in the Active tickbox|


#### Permissions
You will need to grant some permissions (this got things working, and must not be considered best practice)

| Section | Permission | Access level |
|-------|-------|-------|
|Repository permissions|Actions|Read and write|
|Repository permissions|Administration|Read and write|
|Repository permissions|Commit Statuses|Read and write|
|Repository permissions|Contents|Read and write|
|Repository permissions|Metadata *|Read-only|
|Repository permissions|Pull requests|Read and write|
|Organization permissions|Members|Read-only|

* This should be enabled as Mandatory

Click __Create GitHub App__

On the next screen you will be able to see the __App ID__ and __Client ID__. Note these down.

Click the __Generate a new client secret__ button, and copy the secret out for later.

Click the __Generate a private key__ button, this will download a .pem for later.

Finally click the __Save changes__ button.

Use the values above to update the __backstage-secrets.yaml__ file.




### Backstage YARN method

This still needs manual steps though.

```bash
yarn backstage-cli create-github-app blackcatengineering
```
ubuntu@climactic-bull:~/Backstage-Testbed/backstage$ yarn backstage-cli create-github-app blackcatengineering
? Select permissions [required] (these can be changed later but then require approvals in all installations) (Press <space> to select, <a> to toggle all, <i> to invert selection, 
and <enter> to proceed)
 ◉ Read access to content (required by Software Catalog to ingest data from repositories)
 ◉ Read access to members (required by Software Catalog to ingest GitHub teams)
❯◉ Read and Write to content and actions (required by Software Templates to create new repositories)


Pops open your browser 


yarn backstage-cli create-github-app blackcatengineering
? Select permissions [required] (these can be changed later but then require approvals in all installations) Read access to content (required by Software Catalog to 
ingest data from repositories), Read access to members (required by Software Catalog to ingest GitHub teams), Read and Write to content and actions (required by 
Software Templates to create new repositories)
GitHub App configuration written to github-app-backstage-deadbeef-credentials.yaml
This file contains sensitive credentials, it should not be committed to version control and handled with care!
Here's an example on how to update the integrations section in app-config.yaml

integrations:
  github:
    - host: github.com
      apps:


We won't update as it mentions as this would bake the credentials into the app-config.

Instead, use the file to create k8s secrets.

```bash
yq eval '{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {"name": "backstage-github-secrets", "namespace": "backstage"},
  "type": "Opaque",
  "stringData": {
    "GITHUB_APP_ID": .appId | tag == "!!str",
    "GITHUB_APP_CLIENT_ID": .clientId,
    "GITHUB_APP_CLIENT_SECRET": .clientSecret,
    "GITHUB_APP_WEBHOOK_SECRET": .webhookSecret,
    "GITHUB_APP_PRIVATE_KEY": .privateKey
  }
}' github-app-backstage-deadbeef-credentials.yaml > backstage-secrets.yaml
```

Remove the generated file and copy your k8s secret file out of the application directory.
```bash
rm github-app-backstage-deadbeef-credentials.yaml
mv backstage-github-secrets.yaml ..
```

__NOTE__: You will need to setup the GitHub app with the key values and permissions from the manual section.

---

## Might be needed for DB
kubectl exec -it backstage-database-1 -n backstage -- psql -U postgres -d postgres -c "ALTER USER app CREATEDB;"



## Deploy Backstage to your cluster
kubectl apply -f backstage-prod.yaml

What order to apply in?

backstage-ingress.yaml
backstage-prod.yaml
backstage-service.yaml
backstage-issuer.yaml
backstage-redirect.yaml


