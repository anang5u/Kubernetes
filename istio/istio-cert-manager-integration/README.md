# Istio - Integrasi dengan Cert Manager
Sebelum memulai pastikan *tool* berikut telah diinstall di *machine* anda:
- [istioctl](https://github.com/istio/istio/releases/latest)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) and [docker](https://docs.docker.com/get-docker/) (if you're using kind)
- [helm](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [jq](https://stedolan.github.io/jq/download/)

## Creating kubernetes cluster
```
$ kind create cluster --name demo-cluster --config yaml/kind-config.yaml
Creating cluster "demo-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.29.2) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-demo-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-demo-cluster

$ kubectl get nodes
NAME                         STATUS   ROLES           AGE     VERSION
demo-cluster-control-plane   Ready    control-plane   2m44s   v1.29.2
demo-cluster-worker          Ready    <none>          2m20s   v1.29.2
```

## Install cert-manager dengan helm
```
# Helm setup
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update

# install cert-manager CRDs
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.3/cert-manager.crds.yaml

# install cert-manager; this might take a little time
$ helm install cert-manager jetstack/cert-manager \
	--namespace cert-manager \
	--create-namespace \
	--version v1.14.3

# We need this namespace to exist since our cert will be placed there
$ kubectl create namespace istio-system
```
Pastikan cert-manager telah berhasil diinstall
```
# Verify the cert-manager deployments have successfully rolled-out
$ for i in cert-manager cert-manager-cainjector cert-manager-webhook; \
do kubectl rollout status deploy/$i -n cert-manager; done

deployment "cert-manager" successfully rolled out
deployment "cert-manager-cainjector" successfully rolled out
deployment "cert-manager-webhook" successfully rolled out

# Verify cert-manager resources
$ kubectl -n cert-manager get all
NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-6d988558d6-mhqr5              1/1     Running   0          2m25s
pod/cert-manager-cainjector-6976895488-zzc47   1/1     Running   0          2m25s
pod/cert-manager-webhook-fcf48cc54-c8qw8       1/1     Running   0          2m25s

NAME                           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.96.25.63   <none>        9402/TCP   2m25s
service/cert-manager-webhook   ClusterIP   10.96.72.90   <none>        443/TCP    2m25s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           2m25s
deployment.apps/cert-manager-cainjector   1/1     1            1           2m25s
deployment.apps/cert-manager-webhook      1/1     1            1           2m25s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-6d988558d6              1         1         1       2m25s
replicaset.apps/cert-manager-cainjector-6976895488   1         1         1       2m25s
replicaset.apps/cert-manager-webhook-fcf48cc54       1         1         1       2m25s
```

## Create a cert-manager Issuer and Issuing Certificate

```
$ kubectl apply -f yaml/issuer.yaml
issuer.cert-manager.io/selfsigned created
certificate.cert-manager.io/istio-ca created
issuer.cert-manager.io/istio-ca created

$ kubectl get issuers -n istio-system
NAME         READY   AGE
istio-ca     True    2m12s
selfsigned   True    2m12s
```

## Export the Root CA to a Local File
```
# Export our cert from the secret it's stored in, and base64 decode to get the PEM data.
$ kubectl get -n istio-system secret istio-ca -ogo-template='{{index .data "tls.crt"}}' | base64 -d > ca.pem

# Out of interest, we can check out what our CA looks like
$ openssl x509 -in ca.pem -noout -text

# Add our CA to a secret
$ kubectl create secret generic -n cert-manager istio-root-ca --from-file=ca.pem=ca.pem
```

## Install istio-csr
```
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update

# We set a few helm template values so we can point at our static root CA
$ helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr \
	--set "app.tls.rootCAFile=/var/run/secrets/istio-csr/ca.pem" \
	--set "volumeMounts[0].name=root-ca" \
	--set "volumeMounts[0].mountPath=/var/run/secrets/istio-csr" \
	--set "volumes[0].name=root-ca" \
	--set "volumes[0].secret.secretName=istio-root-ca"

NAME: cert-manager-istio-csr
LAST DEPLOYED: Tue Mar  5 09:29:14 2024
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None

# Check to see that the istio-csr pod is running and ready
$  kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6d988558d6-mhqr5              1/1     Running   0          18m
cert-manager-cainjector-6976895488-zzc47   1/1     Running   0          18m
cert-manager-istio-csr-76f6f8c4f-j7qwr     1/1     Running   0          12s
cert-manager-webhook-fcf48cc54-c8qw8       1/1     Running   0          8m

# Certificates istio-ca and istiod should be present and report READY=True
$ kubectl get certificates -n istio-system
NAME       READY   SECRET       AGE
istio-ca   True    istio-ca     25m
istiod     True    istiod-tls   17m
```

## Install Istio
Install `istioctl`
```
$ curl -L https://istio.io/downloadIstio | sh -

$ mv istio-1.20.3/bin/istioctl /usr/local/bin/
$ chmod +x /usr/local/bin/istioctl
$ mv istio-1.20.3 /tmp/

$ istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
  To get started, check out https://istio.io/latest/docs/setup/getting-started/
```

Istio config
```
$ curl -sSL https://raw.githubusercontent.com/cert-manager/website/7f5b2be9dd67831574b9bde2407bed4a920b691c/content/docs/tutorials/istio-csr/example/istio-config-getting-started.yaml > yaml/istio-install-config.yaml
```
Install istio
```
# This takes a little time to complete
$ istioctl install -f yaml/istio-install-config.yaml
This will install the Istio 1.20.3 "demo" profile (with components: Istio core, Istiod, Ingress gateways, and Egress gateways) into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Egress gateways installed
✔ Installation complete
Made this installation the default for injection and validation.
```
Pastikan installasi istio terlah berhasil
```
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-9fbbb6dd9-tnnpf     1/1     Running   0          3m15s
istio-ingressgateway-6fd6df8f65-cxsbh   1/1     Running   0          3m15s
istiod-57df4bdfd6-mbcss                 1/1     Running   0          8m
```

## Validating Install
environment variables
```
# Set namespace for sample application
$ export NAMESPACE=default

# Set env var for the value of the app label in manifests
$ export APP=httpbin

# Grab the installed version of istio
$ export ISTIO_VERSION=$(istioctl version -o json | jq -r '.meshVersion[0].Info.version')
```
Istio injection
```
kubectl label namespace $NAMESPACE istio-injection=enabled --overwrite
```
Apply deployment [httpbin](https://httpbin.org/)
```
$ kubectl create deploy httpbin --image kennethreitz/httpbin
$ kubectl expose deploy httpbin --port 80

$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
httpbin-66c877d7d-8hth4   2/2     Running   0          12m
```
Log `cert-manager`
```
$ kubectl logs -n cert-manager $(kubectl get pods -n cert-manager -o jsonpath='{.items..metadata.name}' --selector app=cert-manager)
$ kubectl logs -n cert-manager $(kubectl get pods -n cert-manager -o jsonpath='{.items..metadata.name}' --selector app=cert-manager) --since 2m -f

I0305 08:44:36.836201       1 conditions.go:263] Setting lastTransitionTime for CertificateRequest "istio-csr-s6zvb" condition "Approved" to 2024-03-05 08:44:36.836189985 +0000 UTC m=+23764.255701884
I0305 08:44:36.880334       1 conditions.go:263] Setting lastTransitionTime for CertificateRequest "istio-csr-s6zvb" condition "Ready" to 2024-03-05 08:44:36.880320075 +0000 UTC m=+23764.299831978
I0305 08:46:28.297673       1 conditions.go:263] Setting lastTransitionTime for CertificateRequest "istio-csr-glzrs" condition "Approved" to 2024-03-05 08:46:28.297662467 +0000 UTC m=+23875.717174362
I0305 08:46:28.322675       1 conditions.go:263] Setting lastTransitionTime for CertificateRequest "istio-csr-glzrs" condition "Ready" to 2024-03-05 08:46:28.322663363 +0000 UTC m=+23875.742175253
```

certificaterequests
```
$ kubectl get certificaterequests.cert-manager.io -n istio-system -w
NAME         APPROVED   DENIED   READY   ISSUER       REQUESTOR                                         AGE
istio-ca-1   True                True    selfsigned   system:serviceaccount:cert-manager:cert-manager   6h22m
istiod-13    True                True    istio-ca     system:serviceaccount:cert-manager:cert-manager   14m
istio-csr-s6zvb                               istio-ca     system:serviceaccount:cert-manager:cert-manager-istio-csr   0s
istio-csr-s6zvb   True                        istio-ca     system:serviceaccount:cert-manager:cert-manager-istio-csr   0s
istio-csr-s6zvb   True                True    istio-ca     system:serviceaccount:cert-manager:cert-manager-istio-csr   0s
istio-csr-s6zvb   True                True    istio-ca     system:serviceaccount:cert-manager:cert-manager-istio-csr   0s
```

```
$ kubectl get certificaterequest -A
NAMESPACE      NAME         APPROVED   DENIED   READY   ISSUER       REQUESTOR                                         AGE
istio-system   istio-ca-1   True                True    selfsigned   system:serviceaccount:cert-manager:cert-manager   6h40m
istio-system   istiod-14    True                True    istio-ca     system:serviceaccount:cert-manager:cert-manager   2m57s

$ kubectl -n istio-system describe certificaterequest istio-ca-1
$ kubectl -n istio-system describe certificaterequest istiod-14
```

- https://cert-manager.io/docs/usage/istio-csr/installation/
- https://docs.tetrate.io/istio-subscription/istio-in-practice/cert-manager-integration