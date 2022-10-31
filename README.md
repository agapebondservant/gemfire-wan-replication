# Running TAP on GKE
* Use instructions here to provision a GKE cluster: 
  * https://medium.com/avmconsulting-blog/kubernetes-google-kubernetes-engine-gke-99abf912f912
  * http://captainvirtualization.com/gke-install-tap/

* To connect to primary cluster:
```
gcloud auth login (if not logged in)
gcloud container clusters get-credentials primary --zone us-east1-b --project pa-oawofolu
(To switch via kubectx: kubectx gke_pa-oawofolu_us-east1-b_primary)
```

*To connect to secondary cluster:
```
gcloud auth login (if not logged in)
gcloud container clusters get-credentials secondary --zone us-west1-a --project pa-oawofolu
(To switch via kubectx: kubectx gke_pa-oawofolu_us-west1-a_secondary)
```

* To generate the kubeconfig file for the secondary cluster:
```
GET_CMD="gcloud container clusters describe secondary --zone=us-west1-a"
```

* Populate a ConfigMap based on the .env file
```
source .env
kubectl delete configmap data-e2e-env || true; sed 's/export //g' .env > .env-properties && kubectl create configmap data-e2e-env --from-env-file=.env-properties && rm .env-properties
```

* Update manifests as appropriate:
```
for orig in `find ../tap -name "*.in.*" -type f`; do
  target=$(echo $orig | sed 's/\.in//')
  envsubst < $orig > $target
  grep -qxF $target .gitignore || echo $target >> .gitignore
done
```

* Create the default storage class (ensure that it is called generic, that the volume binding mode is WaitForFirstCustomer instead of Immediate, and the reclaimPolicy should be Retain)
```
kubectl apply -f ../tap/resources/storageclass-gke.yaml
```

* Mark the storage class as default:
```
kubectl patch storageclass generic -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

* Create the network policy (networkpolicy.yaml - uses allow-all-ingress for now)
```
kubectl apply -f ../tap/resources/networkpolicy.yaml
```

* Ensure that pod scurity policy admission controller is enabled, as PSPs will be created by the eduk8s operator to restrict users from running with root privileges:
```
kubectl apply -f ../tap/resources/podsecuritypolicy.yaml
```

* Install Contour: (NOTE: Change the Loadbalancer's healthcheck from HTTP to TCP if necessary)
```
kubectl apply -f ../tap/resources/contour-gke.yaml
```

* Install the Kubernetes Metrics server:
```
kubectl apply -f ../tap/resources/metrics-server.yaml; watch kubectl get deployment metrics-server -n kube-system
```

* Deploy cluster-scoped cert-manager:
```
kubectl create ns cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
```

* Deploy CERT-MANAGER-ISSUER  (self-signed), CERTIFICATE-SIGNING-REQUEST, CERT-MANAGER-ISSUER (CA):
```
kubectl apply -f ../tap/resources/cert-manager-issuer.yaml
```

* Install SealedSecrets:
```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.4/controller.yaml
```

* Expose Kube-DNS service:
```
kubectl apply -f ../tap/resources/kube-dns-udp.yaml
```

* Install Istio: (used by Multi-site workshops, Gemfire workshops)
```
../tap/istio-1.13.2/bin/istioctl install --set profile=demo -y
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}');
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}');
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}');
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

* Install Gemfire operator:
```
  source .env
  kubectl create ns gemfire-system --dry-run -o yaml | kubectl apply -f -
  kubectl create secret docker-registry image-pull-secret --namespace=gemfire-system --docker-server=registry.pivotal.io --docker-username="$DATA_E2E_PIVOTAL_REGISTRY_USERNAME" --docker-password="$DATA_E2E_PIVOTAL_REGISTRY_PASSWORD" --dry-run -o yaml | kubectl apply -f -
  helm uninstall  gemfire --namespace gemfire-system; helm install gemfire ../tap/other/resources/gemfire/gemfire-operator-1.0.3/ --namespace gemfire-system
```

* Deploy CoreDNS:
```
coredns/deploy.sh | kubectl apply -f -
kubectl delete --namespace=kube-system deployment kube-dns
```

#### Install Operator UI <a name="operatorui"/>
* Install Operator UI: (must have pre-installed Tanzu Data operators)
```
../tap/other/resources/operator-ui/crd_annotations/apply-annotations
kubectl create namespace operator-ui
kubectl create configmap kconfig --from-file </path/to/multicluster/kubeconfig> --namespace operator-ui
kubectl apply -f ../tap/other/resources/operator-ui/tanzu-operator-ui-app.yaml --namespace operator-ui
kubectl apply -f ../tap/other/resources/operator-ui/tanzu-operator-ui-httpproxy.yaml --namespace operator-ui #only if using ProjectContour for Ingress
watch kubectl get all -noperator-ui
```

#### Build secondary cluster (only required for multi-site demo)<a name="multisite"/>
* Create the default storage class (ensure that it is called generic, that the volume binding mode is WaitForFirstCustomer instead of Immediate, and the reclaimPolicy should be Retain):
```    
kubectl apply -f ../tap/resources/storageclass-gke.yaml
kubectl patch storageclass generic -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

* Create the network policy
```
kubectl apply -f ../tap/resources/networkpolicy.yaml
```

* Ensure that pod scurity policy admission controller is enabled, as PSPs will be created by the eduk8s operator to restrict users from running with root privileges:
```    
kubectl apply -f ../tap/resources/podsecuritypolicy.yaml
```

* Install SealedSecrets:
```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.4/controller.yaml
```

* Install the Kubernetes Metrics server:
```
kubectl apply -f ../tap/resources/metrics-server.yaml; watch kubectl get deployment metrics-server -n kube-system
```

* Install cert-manager:
```
kubectl create ns cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
kubectl apply -f ../tap/resources/cert-manager-issuer.yaml
```

* Install Gemfire operator:
```
  source .env
  kubectl create ns gemfire-system --dry-run -o yaml | kubectl apply -f -
  kubectl create ns gemfire-remote --dry-run -o yaml | kubectl apply -f -
  kubectl create secret docker-registry image-pull-secret --namespace=gemfire-remote --docker-server=registry.pivotal.io --docker-username="$DATA_E2E_PIVOTAL_REGISTRY_USERNAME" --docker-password="$DATA_E2E_PIVOTAL_REGISTRY_PASSWORD"
  kubectl create secret docker-registry image-pull-secret --namespace=gemfire-system --docker-server=registry.pivotal.io --docker-username="$DATA_E2E_PIVOTAL_REGISTRY_USERNAME" --docker-password="$DATA_E2E_PIVOTAL_REGISTRY_PASSWORD" --dry-run -o yaml | kubectl apply -f -
  helm uninstall  gemfire --namespace gemfire-system; helm install gemfire ../tap/other/resources/gemfire/gemfire-operator-1.0.3/ --namespace gemfire-system
```

* Expose Kube-DNS service:
```
kubectl apply -f ../tap/resources/kube-dns-udp.yaml
```

* Deploy CoreDNS:
```
coredns/deploy.sh | kubectl apply -f -
kubectl delete --namespace=kube-system deployment kube-dns
```

* Install Istio:
- (In primary cluster)
```
../tap/istio-1.13.2/bin/istioctl install --set profile=demo-tanzu-gcp --set installPackagePath=../tap/istio-1.13.2/manifests -y --set values.gateways.istio-ingressgateway.runAsRoot=true
```
- (In secondary cluster)
```
../tap/istio-1.13.2/bin/istioctl install --set profile=demo-tanzu-gcp --set installPackagePath=../tap/istio-1.13.2/manifests -y --set values.gateways.istio-ingressgateway.runAsRoot=true 
```

* Generate kubeconfig for accessing secondary cluster: (Must install kubeseal: https://github.com/bitnami-labs/sealed-secrets)
  (On secondary cluster:)
```
kubectl create secret generic kconfig --from-file=myfile --dry-run=client -o yaml > kconfigsecret.yaml
```

(On primary cluster:)
```
kubectl apply -f kconfigsecret.yaml
```

### Install TAP<a name="tap-install"/>
(NOTE: TAP pre-reqs: https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-install-intro.html)

#### Install TAP command line tooling
```
mkdir $HOME/tanzu
export TANZU_CLI_NO_INIT=true
cd $HOME/tanzu
sudo install cli/core/0.11.1/tanzu-core-linux_amd64 /usr/local/bin/tanzu
tanzu plugin install --local cli all
tanzu plugin list
```
##### Install imgpkg
```
wget -O- https://carvel.dev/install.sh > install.sh
sudo bash install.sh
imgpkg version
```
#### Relocate images to local registry
```
source .env
docker login registry-1.docker.io
docker login registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=$DATA_E2E_REGISTRY_USERNAME
export INSTALL_REGISTRY_PASSWORD=$DATA_E2E_REGISTRY_PASSWORD
#export TAP_VERSION=1.1.1
export TAP_VERSION=1.2.0
export INSTALL_REGISTRY_HOSTNAME=index.docker.io
imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${TAP_VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/${DATA_E2E_REGISTRY_USERNAME}/tap-packages
imgpkg copy -b registry.tanzu.vmware.com/p-rabbitmq-for-kubernetes/tanzu-rabbitmq-package-repo:${DATA_E2E_RABBIT_OPERATOR_VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/oawofolu/vmware-tanzu-rabbitmq
```

#### Install kappcontroller, secretgen
```
export INSTALL_BUNDLE=registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:82dfaf70656b54dcba0d4def85ccae1578ff27054e7533d08320244af7fb0343
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=$DATA_E2E_PIVOTAL_REGISTRY_USERNAME
export INSTALL_REGISTRY_PASSWORD=$DATA_E2E_PIVOTAL_REGISTRY_PASSWORD
export TAP_VERSION=1.2.0
cd tanzu-cluster-essentials-darwin-amd64-1.3.0
./install.sh
```

#### Install TAP
```
kubectl create ns tap-install


export INSTALL_REGISTRY_HOSTNAME=index.docker.io
export INSTALL_REGISTRY_USERNAME=$DATA_E2E_REGISTRY_USERNAME
export INSTALL_REGISTRY_PASSWORD=$DATA_E2E_REGISTRY_PASSWORD

tanzu secret registry add tap-registry \
--username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} \
--server ${INSTALL_REGISTRY_HOSTNAME} \
--export-to-all-namespaces --yes --namespace tap-install

tanzu package repository add tanzu-tap-repository \
--url ${INSTALL_REGISTRY_HOSTNAME}/${DATA_E2E_REGISTRY_USERNAME}/tap-packages:$TAP_VERSION \
--namespace tap-install


export INSTALL_REGISTRY_HOSTNAME=index.docker.io
tanzu package repository get tanzu-tap-repository --namespace tap-install
tanzu package available list --namespace tap-install
tanzu package available list tap.tanzu.vmware.com --namespace tap-install
tanzu package available get tap.tanzu.vmware.com/$TAP_VERSION --values-schema --namespace tap-install
tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file ../tap/resources/tap-values.yaml -n tap-install
```

To check on a package's install status:
```
tanzu package installed get tap<or name pf package> -n tap-install
```

To check that all expected packages were installed successfully:
```
tanzu package installed list -A -n tap-install
```

* Install Learning Center:
```
tanzu package available list learningcenter.tanzu.vmware.com --namespace tap-install # To view available packages for learningcenter
tanzu package install learning-center --package-name learningcenter.tanzu.vmware.com --version 0.2.1 -f ../tap/resources/learning-center-config.yaml -n tap-install
kubectl get all -n learningcenter
tanzu package available list workshops.learningcenter.tanzu.vmware.com --namespace tap-install
```

##### Deploy "WAN Replication workshop"<a name="workshopa"/>
Add the following to your `training-portal.yaml` (under **spec.workshops**):
```
- name: data-gemfire-wan-replication
    capacity: 10 #Change the capacity to the number of expected participants
    capacity: 5
    expires: 120m
    orphaned: 5m
```

Run the following:
```
kubectl delete --all learningcenter-training
cd ../tap
resources/scripts/deploy-handson-workshop.sh  ../gcloud/.env
cd ../gcloud
kubectl apply -f ../tap/resources/hands-on/system-profile.yaml
kubectl apply -f ../tap/resources/hands-on/workshop-gemfire-wan-replication-demo.yaml
kubectl apply -f ../tap/resources/hands-on/training-portal-sample.yaml
watch kubectl get learningcenter-training
```