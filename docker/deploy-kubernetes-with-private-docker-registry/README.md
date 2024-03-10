# Deploy Image from Private Docker Registry into Kubernetes

At this time, we already have private docker registry *runing* at `registry.anangsu13.cloud`

for more about how create private docker images registry (self hosted), authenticate and docker registry UI

https://github.com/anang5u/kubernetes/tree/master/docker/authenticate-proxy-with-nginx

## Kubernetes cluster
```
$ kubectl get nodes
NAME                         STATUS   ROLES           AGE   VERSION
demo-cluster-control-plane   Ready    control-plane   2d   v1.29.2
demo-cluster-worker          Ready    <none>          2d   v1.29.2
```

## Deploy nginx 
Deploy `nginx` image from private registry `registry.anangsu13.cloud/nginx:latest`

## Deploy without credentials
```
$ kubectl create deploy mytest --image=registry.anangsu13.cloud/nginx:latest
deployment.apps/mytest created

$ kubectl get pod
NAME                                 READY   STATUS             RESTARTS       AGE
mytest-67465c9888-27rl7              0/1     ImagePullBackOff   0              8s

$ kubectl describe pod/mytest-67465c9888-27rl7 | less
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Warning  Failed     54s (x5 over 83s)  kubelet            Error: ImagePullBackOff
  Normal   Pulling    41s (x3 over 84s)  kubelet            Pulling image "registry.anangsu13.cloud/nginx:latest"
  Warning  Failed     41s (x3 over 84s)  kubelet            Failed to pull image "registry.anangsu13.cloud/nginx:latest": failed to pull and unpack image "registry.anangsu13.cloud/nginx:latest": failed to resolve reference "registry.anangsu13.cloud/nginx:latest": pull access denied, repository does not exist or may require authorization: authorization failed: no basic auth credentials
  Warning  Failed     41s (x3 over 84s)  kubelet            Error: ErrImagePull

# kubectl delete deploy mytest
deployment.apps "mytest" deleted
```
`pull access denied, repository does not exist or may require authorization: authorization failed: no basic auth credentials`

## Create Secret
Doker registry `registry.anangsu13.cloud` with username `testuser` and password `testpassword`
```
$ kubectl create secret docker-registry mydockercredentials \
--docker-server registry.anangsu13.cloud \
--docker-username testuser  \
--docker-password testpassword

$ kubectl describe secret mydockercredentials
Name:         mydockercredentials
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  126 bytes
```

## Create Deployment file
```
$ kubectl create deploy mytest --image=registry.anangsu13.cloud/nginx:latest --dry-run=client -o yaml > /tmp/mytest.yaml 
```

Modify yaml file `/tmp/mytest.yaml`
```javascript
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: mytest
  name: mytest
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mytest
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mytest
    spec:
      containers:
      - image: registry.anangsu13.cloud/nginx:latest
        imagePullPolicy: Always
        name: nginx
      imagePullSecrets:
      - name: mydockercredentials
```

## Deploy with credentials
```
$ kubectl apply -f /tmp/mytest.yaml
deployment.apps/mytest created

$ kubectl get pod
NAME                                 READY   STATUS    RESTARTS        AGE
mytest-7cd9f549c4-m6257              1/1     Running   0               102s

$ kubectl watch events
0s                  Normal    ScalingReplicaSet   Deployment/mytest              Scaled up replica set mytest-7cd9f549c4 to 1
0s                  Normal    SuccessfulCreate    ReplicaSet/mytest-7cd9f549c4   Created pod: mytest-7cd9f549c4-m6257
0s                  Normal    Scheduled           Pod/mytest-7cd9f549c4-m6257    Successfully assigned default/mytest-7cd9f549c4-m6257 to demo-cluster-worker
0s                  Normal    Pulled              Pod/mytest-7cd9f549c4-m6257    Container image "docker.io/istio/proxyv2:1.20.3" already present on machine
0s                  Normal    Pulling             Pod/mytest-7cd9f549c4-m6257    Pulling image "registry.anangsu13.cloud/nginx:latest"
0s                  Normal    Pulled              Pod/mytest-7cd9f549c4-m6257    Successfully pulled image "registry.anangsu13.cloud/nginx:latest" in 663ms (663ms including waiting)
0s                  Normal    Created             Pod/mytest-7cd9f549c4-m6257    Created container nginx
0s                  Normal    Started             Pod/mytest-7cd9f549c4-m6257    Started container nginx
```
