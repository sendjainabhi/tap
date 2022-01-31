# Tanzu Application Platform - Deployment


## Executive Summary
This deployment guide outlines a deployment steps of VMware Tanzu Application Platform on Kubernetes workload cluster. 
## Prerequisites
Before deploying VMware Tanzu Application Platform , ensure that the following are set up.

* A [Tanzu Network](https://network.tanzu.vmware.com/) account to download Tanzu Application Platform packages.
* A container image registry access with push and write access, such as Harbor , Docker Hub or Azure Container Registry for application images, base images, and runtime dependencies.
* Network access to [VMware registery](https://registry.tanzu.vmware.com)
* DNS Records for components like  Cloud Native Runtimes (knative) i.e cnrs , Tanzu Learning Center , Tanzu Application Platform  GUI etc. 
* A Git repository from GitHub ,Gitlab or Azure DevOps for the Tanzu Application Platform GUI's software catalogs, along with a token allowing read access.
* Kubernetes workload cluster versions 1.20, 1.21, or 1.22 on Azure Kubernetes Service or Amazon Elastic Kubernetes Service or 
Google Kubernetes Engine or Minikube Kubernetes providers.
* Accept the End User License Agreements (EULAs).
* The Kubernetes CLI, kubectl, v1.20, v1.21 or v1.22, installed and authenticated with administrator rights for your target cluster.

You can refer further details of [Prerequisites](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install-general.html#prereqs).


## Overview of the Deployment Steps

The following provides an overview of the major steps necessary to deploy Tanzu Application Platform. Each steps links to the section for detailed information.


1. [Setup Tanzu Application Platform Build cluster](#tap-build)
2. [Setup Tanzu Application Platform Run cluster](#tap-run)
3. [Setup Tanzu Application Platform UI cluster](#tap-ui)
4. [Setup Tanzu Application Platform Full(personal) cluster](#tap-full)


 [Install tanzu cluster essentials and tanzu cli](#tanzu-essential)



## <a id=tap-build> </a> Setup Tanzu Application Platform Build cluster

### <a id=tanzu-essential> </a> Step 1: Install tanzu cluster essentials and tanzu cli

Provide following user inputs into commands and execute them to install tanzu cluster essentials and tanzu cli into bootstrap/jumpbox machine. 

* Tanzu-Net-API-Token(refresh_token)
* TANZU-NET-USER 
* TANZU-NET-PASSWORD

```
# login to kubernetes workload cluster using cluster config
kubectl config get-contexts
kubectl config use-context <cluster config name>

#login to Tanzu net (https://network.tanzu.vmware.com/). And get API token from profile page.
export refresh_token=<Tanzu-Net-API-Token>
export token=$(curl -X POST https://network.pivotal.io/api/v2/authentication/access_tokens -d '{"refresh_token":"'${refresh_token}'"}')
access_token=$(echo ${token} | jq -r .access_token)

curl -i -H "Accept: application/json" -H "Content-Type: application/json" -H "Authorization: Bearer ${access_token}" -X GET https://network.pivotal.io/api/v2/authentication

#install tanzu cluster essential(linux)
mkdir $HOME/tanzu-cluster-essentials
wget https://network.tanzu.vmware.com/api/v2/products/tanzu-cluster-essentials/releases/1011100/product_files/1105818/download --header="Authorization: Bearer ${access_token}" -O $HOME/tanzu-cluster-essentials/tanzu-cluster-essentials-linux-amd64-1.0.0.tgz
tar -xvf $HOME/tanzu-cluster-essentials/tanzu-cluster-essentials-linux-amd64-1.0.0.tgz -C $HOME/tanzu-cluster-essentials


export INSTALL_BUNDLE=registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:82dfaf70656b54dcba0d4def85ccae1578ff27054e7533d08320244af7fb0343
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=<TANZU-NET-USER>
export INSTALL_REGISTRY_PASSWORD=<TANZU-NET-PASSWORD>
cd $HOME/tanzu-cluster-essentials
./install.sh

sudo cp $HOME/tanzu-cluster-essentials/kapp /usr/local/bin/kapp

cd $HOME

#install tanzu cli v(0.10.0) and plug-ins (linux)
mkdir $HOME/tanzu
cd $HOME/tanzu
wget https://network.pivotal.io/api/v2/products/tanzu-application-platform/releases/1030465/product_files/1114447/download --header="Authorization: Bearer ${access_token}" -O $HOME/tanzu/tanzu-framework-linux-amd64.tar
tar -xvf $HOME/tanzu/tanzu-framework-linux-amd64.tar -C $HOME/tanzu

sudo install cli/core/v0.10.0/tanzu-core-linux_amd64 /usr/local/bin/tanzu

#tanzu plug-ins
export TANZU_CLI_NO_INIT=true
tanzu plugin install --local cli all
tanzu plugin list

```

### <a id=tap-package-repo> </a>Step 2: Add the Tanzu Application Platform package repository

Execute following commands to add TAP package. 

```

kubectl create ns tap-install

#tanzu registry secret creation
tanzu secret registry add tap-registry \
  --username "${INSTALL_REGISTRY_USERNAME}" --password "${INSTALL_REGISTRY_PASSWORD}" \
  --server "${INSTALL_REGISTRY_HOSTNAME}" \
  --export-to-all-namespaces --yes --namespace tap-install

#tanzu repo add
tanzu package repository add tanzu-tap-repository \
  --url registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.0.0 \
  --namespace tap-install

tanzu package repository get tanzu-tap-repository --namespace tap-install

#tap available package list
tanzu package available list --namespace tap-install

```

### <a id=tap-profile-build> </a>Step 3: Install Tanzu Application Platform build profile

Provide following user inputs to set environments variables into commands and execute them to install build profile

* registry_server - uri of registry server like Azure container registry or Harbor etc. (example - tappoc.azurecr.io)
* registry_user - registry server user
* registry_password - registry server user


```
export registry_server=<registry server uri>
export registry_user=<registry_user>
export registry_password=<registry_password>

#APPEND GUI SETTINGS
rm tap-values.yaml
cat <<EOF | tee tap-values.yaml
profile: full
ceip_policy_disclosed: true
buildservice:
  kp_default_repository: $registry_server/build-service
  kp_default_repository_username: $registry_user
  kp_default_repository_password: $registry_password
  tanzunet_username: $INSTALL_REGISTRY_USERNAME
  tanzunet_password: $INSTALL_REGISTRY_PASSWORD
supply_chain: basic
ootb_supply_chain_basic:
  registry:
    server: $registry_server
    repository: "supply-chain"
  gitops:
    ssh_secret: ""
  cluster_builder: default
  service_account: default
tap_gui:
  service_type: LoadBalancer
  
metadata_store:
  app_service_type: LoadBalancer

excluded_packages:
  - accelerator.apps.tanzu.vmware.com
  - run.appliveview.tanzu.vmware.com
  - api-portal.tanzu.vmware.com
  - cnrs.tanzu.vmware.com
  - ootb-delivery-basic.tanzu.vmware.com
  - developer-conventions.tanzu.vmware.com
  - image-policy-webhook.signing.apps.tanzu.vmware.com
  - learningcenter.tanzu.vmware.com
  - workshops.learningcenter.tanzu.vmware.com
  - services-toolkit.tanzu.vmware.com
  - service-bindings.labs.vmware.com
  - tap-gui.tanzu.vmware.com

EOF

tanzu package install tap -p tap.tanzu.vmware.com -v 1.0.0 --values-file tap-values.yaml -n tap-install
tanzu package installed get tap -n tap-install

#check all build cluster package installed succesfully
tanzu package installed list -A

```


### <a id=tap-dev-namespace> </a>Step 4: Set up developer namespaces to use installed packages

Execute following commands to setup developer namespaces to use installed packages.

```
export namespace=default

tanzu secret registry add registry-credentials --server "${registry_server}" --username "${registry_user}" --password "${registry_password}" --namespace $namespace


cat <<EOF | kubectl -n default apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: tap-registry
  annotations:
    secretgen.carvel.dev/image-pull-secret: ""
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
secrets:
  - name: registry-credentials
imagePullSecrets:
  - name: registry-credentials
  - name: tap-registry

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: default
rules:
- apiGroups: [source.toolkit.fluxcd.io]
  resources: [gitrepositories]
  verbs: ['*']
- apiGroups: [source.apps.tanzu.vmware.com]
  resources: [imagerepositories]
  verbs: ['*']
- apiGroups: [carto.run]
  resources: [deliverables, runnables]
  verbs: ['*']
- apiGroups: [kpack.io]
  resources: [images]
  verbs: ['*']
- apiGroups: [conventions.apps.tanzu.vmware.com]
  resources: [podintents]
  verbs: ['*']
- apiGroups: [""]
  resources: ['configmaps']
  verbs: ['*']
- apiGroups: [""]
  resources: ['pods']
  verbs: ['list']
- apiGroups: [tekton.dev]
  resources: [taskruns, pipelineruns]
  verbs: ['*']
- apiGroups: [tekton.dev]
  resources: [pipelines]
  verbs: ['list']
- apiGroups: [kappctrl.k14s.io]
  resources: [apps]
  verbs: ['*']
- apiGroups: [serving.knative.dev]
  resources: ['services']
  verbs: ['*']
- apiGroups: [servicebinding.io]
  resources: ['servicebindings']
  verbs: ['*']
- apiGroups: [services.apps.tanzu.vmware.com]
  resources: ['resourceclaims']
  verbs: ['*']
- apiGroups: [scanning.apps.tanzu.vmware.com]
  resources: ['imagescans', 'sourcescans']
  verbs: ['*']

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: default
subjects:
  - kind: ServiceAccount
    name: default

EOF
```



## <a id=tap-run> </a> Setup Tanzu Application Platform Run cluster

### Step 1: Install tanzu cluster essentials and tanzu cli - 
Please ensure you login into Kubernetes run cluster and perform steps of [Install tanzu cluster essentials and tanzu cli](#tanzu-essential). 

### Step 2: Add the Tanzu Application Platform package repository - 
Perform steps of [Add the Tanzu Application Platform package repository](#tap-package-repo)


### <a id=tap-profile-run> </a>Step 3: Install Tanzu Application Platform run profile

Provide following user inputs to set environments variables into commands and execute them to install build profile

* registry_server - uri of registry server like Azure container registry or Harbor etc. (example - tappoc.azurecr.io)
* registry_user - registry server user
* registry_password - registry server user


```
export registry_server=<registry server uri>
export registry_user=<registry_user>
export registry_password=<registry_password>

#APPEND GUI SETTINGS
rm tap-values.yaml
cat <<EOF | tee tap-values.yaml
profile: full
ceip_policy_disclosed: true
buildservice:
  kp_default_repository: $registry_server/build-service
  kp_default_repository_username: $registry_user
  kp_default_repository_password: $registry_password
  tanzunet_username: $INSTALL_REGISTRY_USERNAME
  tanzunet_password: $INSTALL_REGISTRY_PASSWORD
supply_chain: basic
ootb_supply_chain_basic:
  registry:
    server: $registry_server
    repository: "supply-chain"
  gitops:
    ssh_secret: ""
  cluster_builder: default
  service_account: default
tap_gui:
  service_type: LoadBalancer
  
metadata_store:
  app_service_type: LoadBalancer

excluded_packages:
  - accelerator.apps.tanzu.vmware.com
  - run.appliveview.tanzu.vmware.com
  - api-portal.tanzu.vmware.com
  - cnrs.tanzu.vmware.com
  - ootb-delivery-basic.tanzu.vmware.com
  - developer-conventions.tanzu.vmware.com
  - image-policy-webhook.signing.apps.tanzu.vmware.com
  - learningcenter.tanzu.vmware.com
  - workshops.learningcenter.tanzu.vmware.com
  - services-toolkit.tanzu.vmware.com
  - service-bindings.labs.vmware.com
  - tap-gui.tanzu.vmware.com

EOF

tanzu package install tap -p tap.tanzu.vmware.com -v 1.0.0 --values-file tap-values.yaml -n tap-install
tanzu package installed get tap -n tap-install

#check all build cluster package installed succesfully
tanzu package installed list -A

```
 
### Step 4: Set up developer namespaces to use installed packages 
Perform steps of [Set up developer namespaces to use installed packages](#tap-dev-namespace)

You can execute above Steps 1-4 of [Setup Tanzu Application Platform Run cluster](#tap-run) to build Dev/Test/QA/Prod clusters. 


## <a id=tap-ui> </a> Setup Tanzu Application Platform UI cluster

### Step 1: Install tanzu cluster essentials and tanzu cli - 
Please ensure you login into Kubernetes UI cluster and perform steps of [Install tanzu cluster essentials and tanzu cli](#tanzu-essential). 

### Step 2: Add the Tanzu Application Platform package repository - 
Perform steps of [Add the Tanzu Application Platform package repository](#tap-package-repo)


### <a id=tap-profile-ui> </a>Step 3: Install Tanzu Application Platform ui profile

Provide following user inputs to set environments variables into commands and execute them to install build profile

* registry_server - uri of registry server like Azure container registry or Harbor etc. (example - tappoc.azurecr.io)
* registry_user - registry server user
* registry_password - registry server user
* github_token - git hub account token.
* app_domain  - app domain you want to use for tap-gui
* git_catalog_url - git catelog url. if you don't have one , use this [example](https://github.com/sendjainabhi/tap/blob/main/catalog-info.yaml)

 Refer [full profile](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install.html#full-profile) documentation for further details. 


```
#set following variablels
export registry_server=<registry server uri>
export registry_user=<registry_user>
export registry_password=<registry_password>
export github_token=<github_token>
export app_domain=<app_domain>
export git_catalog_url=<git_catalog_url>


#APPEND GUI SETTINGS
rm tap-values.yaml
cat <<EOF | tee tap-values.yaml
profile: full
ceip_policy_disclosed: true
buildservice:
  kp_default_repository: $registry_server/build-service
  kp_default_repository_username: $registry_user
  kp_default_repository_password: $registry_password
  tanzunet_username: $INSTALL_REGISTRY_USERNAME
  tanzunet_password: $INSTALL_REGISTRY_PASSWORD
supply_chain: basic
ootb_supply_chain_basic:
  registry:
    server: $registry_server
    repository: "supply-chain"
  gitops:
    ssh_secret: ""
  cluster_builder: default
  service_account: default

tap_gui:
  service_type: LoadBalancer
  ingressEnabled: "true"
  ingressDomain: "${app_domain}"
  app_config:
    app:
      baseUrl: http://tap-gui.${app_domain}
    catalog:
      locations:
        - type: url
          target: $git_catalog_url/catalog-info.yaml
    backend:
        baseUrl: http://tap-gui.${app_domain}
        cors:
          origin: http://tap-gui.${app_domain}
    integrations:
      github:
        - host: github.com
          token: $github_token
contour:
  envoy:
    service:
      type: LoadBalancer
cnrs:
  domain_name: $app_domain

metadata_store:
  app_service_type: LoadBalancer

excluded_packages:
  - accelerator.apps.tanzu.vmware.com
  - api-portal.tanzu.vmware.com
  - build.appliveview.tanzu.vmware.com
  - buildservice.tanzu.vmware.com
  - controller.conventions.apps.tanzu.vmware.com
  - developer-conventions.tanzu.vmware.com
  - grype.scanning.apps.tanzu.vmware.com
  - learningcenter.tanzu.vmware.com
  - metadata-store.apps.tanzu.vmware.com
  - ootb-supply-chain-basic.tanzu.vmware.com
  - ootb-supply-chain-testing.tanzu.vmware.com
  - ootb-supply-chain-testing-scanning.tanzu.vmware.com
  - scanning.apps.tanzu.vmware.com
  - spring-boot-conventions.tanzu.vmware.com
  - tap-gui.tanzu.vmware.com
  - workshops.learningcenter.tanzu.vmware.com

EOF

tanzu package install tap -p tap.tanzu.vmware.com -v 1.0.0 --values-file tap-values.yaml -n tap-install
tanzu package installed get tap -n tap-install

#check all build cluster package installed succesfully
tanzu package installed list -A

kubectl get svc -n tanzu-system-ingress

#pick external ip from output and configure DNS wild card into your DNS server like aws route 53 etc. 

```
 
### Step 4: Set up developer namespaces to use installed packages 
Perform steps of [Set up developer namespaces to use installed packages](#tap-dev-namespace)

## <a id=tap-sample-app> Deploy Sample application

Execute following command to see the demo of sample app deployment into Tanzu Application Platform

```

export app_name=tap-demo
export git_app_url=https://github.com/sample-accelerators/spring-petclinic

tanzu apps workload delete --all

tanzu apps workload list

#generate work load yml file
tanzu apps workload create ${app_name} --git-repo ${git_app_url} --git-branch main --type web --label app.kubernetes.io/part-of=${app_name} --yes --dry-run

#create work load for app
tanzu apps workload create ${app_name} --git-repo ${git_app_url} --git-branch main --type web --label app.kubernetes.io/part-of=${app_name} --yes

#app deployment logs
tanzu apps workload tail ${app_name} --since 10m --timestamp

#get app workload list
tanzu apps workload list

#get app details
tanzu apps workload get ${app_name}

#copy  app url and paste into browser to see the sample app

```


## <a id=tap-full> </a> Setup Tanzu Application Platform personal(full) cluster

### Step 1: Install tanzu cluster essentials and tanzu cli - 
Please ensure you login into Kubernetes UI cluster and perform steps of [Install tanzu cluster essentials and tanzu cli](#tanzu-essential). 

### Step 2: Add the Tanzu Application Platform package repository - 
Perform steps of [Add the Tanzu Application Platform package repository](#tap-package-repo)


### <a id=tap-profile-full> </a>Step 3: Install Tanzu Application Platform personal(full) profile

Provide following user inputs to set environments variables into commands and execute them to install build profile

* registry_server - uri of registry server like Azure container registry or Harbor etc. (example - tappoc.azurecr.io)
* registry_user - registry server user
* registry_password - registry server user
* github_token - git hub account token.
* app_domain  - app domain you want to use for tap-gui
* git_catalog_url - git catelog url. if you don't have one , use this [example](https://github.com/sendjainabhi/tap/blob/main/catalog-info.yaml)

 Refer [full profile](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install.html#full-profile) documentation for further details. 


```
#set following variablels
export registry_server=<registry server uri>
export registry_user=<registry_user>
export registry_password=<registry_password>
export github_token=<github_token>
export app_domain=<app_domain>
export git_catalog_url=<git_catalog_url>


#APPEND GUI SETTINGS
rm tap-values.yaml
cat <<EOF | tee tap-values.yaml
profile: full
ceip_policy_disclosed: true
buildservice:
  kp_default_repository: $registry_server/build-service
  kp_default_repository_username: $registry_user
  kp_default_repository_password: $registry_password
  tanzunet_username: $INSTALL_REGISTRY_USERNAME
  tanzunet_password: $INSTALL_REGISTRY_PASSWORD
supply_chain: basic
ootb_supply_chain_basic:
  registry:
    server: $registry_server
    repository: "supply-chain"
  gitops:
    ssh_secret: ""
  cluster_builder: default
  service_account: default

tap_gui:
  service_type: LoadBalancer
  ingressEnabled: "true"
  ingressDomain: "${app_domain}"
  app_config:
    app:
      baseUrl: http://tap-gui.${app_domain}
    catalog:
      locations:
        - type: url
          target: $git_catalog_url/catalog-info.yaml
    backend:
        baseUrl: http://tap-gui.${app_domain}
        cors:
          origin: http://tap-gui.${app_domain}
    integrations:
      github:
        - host: github.com
          token: $github_token
contour:
  envoy:
    service:
      type: LoadBalancer
cnrs:
  domain_name: $app_domain

metadata_store:
  app_service_type: LoadBalancer

EOF

tanzu package install tap -p tap.tanzu.vmware.com -v 1.0.0 --values-file tap-values.yaml -n tap-install
tanzu package installed get tap -n tap-install

#check all build cluster package installed succesfully
tanzu package installed list -A

kubectl get svc -n tanzu-system-ingress

#pick external ip from output and configure DNS wild card into your DNS server like aws route 53 etc. 

```
 
### Step 4: Set up developer namespaces to use installed packages 
Perform steps of [Set up developer namespaces to use installed packages](#tap-dev-namespace)

### Deploy Sample application 
See the steps to deploy and test [sample application](#tap-sample-app). You can refer [Deploy Application documentation](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-getting-started.html) for further details.