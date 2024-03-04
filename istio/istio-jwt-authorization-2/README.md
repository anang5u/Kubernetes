# Istio - JWT Authorization #2
Sebelumnya telah mencoba *deploy* auth microservice [Istio - Authorization Policy dengan JWT](https://github.com/anang5u/Kubernetes/tree/master/istio/jwt-token). Disini akan mencoba kembali bagaimana istio melakukan proteksi terhadap microservice yang ada didalam sebuah cluster, istio akan meneruskan jwt claims ke microservice lainnya dalam bentuk *header*

Sebelum memaulai, silahkan untuk mengikuti petunjuk [Istio - Install Istio-Service Mesh dengan Istioctl, Gateway, Virtual Service](https://github.com/anang5u/Kubernetes/tree/master/istio/istio-kubernetes-in-docker-kind)

```
$ kubectl get pod,svc,gw,vs
NAME                                     READY   STATUS    RESTARTS   AGE
pod/httpbin-66c877d7d-dtf5j              2/2     Running   0          89m

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/httpbin             ClusterIP   10.96.33.126    <none>        80/TCP     98m
service/kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP    3h59m

NAME                                               AGE
gateway.networking.istio.io/istio-ingressgateway   92m

NAME                                             GATEWAYS                   HOSTS   AGE
virtualservice.networking.istio.io/demo-routes   ["istio-ingressgateway"]   ["*"]   92m

$ kubectl -n istio-system get svc
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.96.172.130   172.18.0.241   15021:31946/TCP,80:32042/TCP,443:32224/TCP   167m
istiod                 ClusterIP      10.96.72.70     <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP        169m
```
## Akses Microservice httpbin tanpa Authorization
```
$ curl 172.18.0.241/httpbin/get
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "172.18.0.241",
    "User-Agent": "curl/7.81.0",
    "X-B3-Parentspanid": "fd8e8b3e2bb34ac9",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "1872d944660076af",
    "X-B3-Traceid": "33bc4ff4833a5f1bfd8e8b3e2bb34ac9",
    "X-Envoy-Attempt-Count": "1",
    "X-Envoy-Internal": "true",
    "X-Envoy-Original-Path": "/httpbin/get",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=22feb51ffec470cf6f2af2d8a910e2e3481fc149bcbe2d9f51b375a1105a5e40;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  },
  "origin": "10.244.1.1",
  "url": "http://172.18.0.241/get"
}
```

## Deploy Auth Microservice
```
$ kubectl create deploy auth-microservice --image anangsu13/jwt-auth-microservice:v4.3
$ kubectl expose deploy auth-microservice --port 8080

$ kubectl get pod,svc
NAME                                     READY   STATUS    RESTARTS   AGE
pod/auth-microservice-79fccf97f4-tcjvh   2/2     Running   0          37m
pod/httpbin-66c877d7d-dtf5j              2/2     Running   0          97m

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/auth-microservice   ClusterIP   10.96.104.249   <none>        8080/TCP   35m
service/httpbin             ClusterIP   10.96.33.126    <none>        80/TCP     106m
service/kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP    3h7m
```
## Apply auth-routes
```
$ kubectl apply -f yaml/istio-gateway.yaml

$ curl -X POST 172.18.0.241/auth/v1/login
{"token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJjb21wYW55IjoiVk9CTWNOQmJNeiIsImlhdCI6NTAwLCJpc3MiOiJkZW1vLWFwaUBpc3N1ZXIuY2xvdWQiLCJzdWIiOiJqd3RAZXhhbXBsZS5jb20iLCJ1c2VyaWQiOiJ1ZU9zUFBTbEhOIn0.PRzNpWY6-K_l0-GtEZ6CUvn6CKLTw_SmimZGXYlZCgwy6y5TpfVFkipqEI1vOnjn2WvuSVefeDasZgXWCE91UedEFfF-ruxGuLQEEClU5d8rrnBoZo0ivD8gXNuIP4XV2zDHVRzRcfY-eVFTRO8duQ8bzsPMmxye6u1IqhG2MXlkz6EKRgx9rNf8D5a3rmdnPFOjTe-RLNYlV3rGbMx6g-YBQ9WX79GRV2YnhElyKXorzyV-gFJIhWNdopdb7U_U0kNxyAL7IW2b1U8W0oOo9qfQo7C1oIqv2CxZwjiAuA-C0vddV2yfy42SNTAX0Gho0eWQuTEwbegbIjaPjypV7g"}
```

## Deploy JWT Authorization - JWT Token
https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/

```
$ kubectl apply -f Kubernetes/istio-kind/yaml/jwt-authorization.yaml
requestauthentication.security.istio.io/jwt-on-ingress created
authorizationpolicy.security.istio.io/require-jwt created
```

## Akses Microservice httpbin dengan Bearer Token JWT
### Mencoba akses tanpa token
```
$ curl 172.18.0.241/httpbin/get
RBAC: access denied

$ curl 172.18.0.241/httpbin/get -sS -o /dev/null -w "%{http_code}\n"
403
```
### Akses dengan token
```
$ TOKEN=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJjb21wYW55IjoiVk9CTWNOQmJNeiIsImlhdCI6NTAwLCJpc3MiOiJkZW1vLWFwaUBpc3N1ZXIuY2xvdWQiLCJzdWIiOiJqd3RAZXhhbXBsZS5jb20iLCJ1c2VyaWQiOiJ1ZU9zUFBTbEhOIn0.PRzNpWY6-K_l0-GtEZ6CUvn6CKLTw_SmimZGXYlZCgwy6y5TpfVFkipqEI1vOnjn2WvuSVefeDasZgXWCE91UedEFfF-ruxGuLQEEClU5d8rrnBoZo0ivD8gXNuIP4XV2zDHVRzRcfY-eVFTRO8duQ8bzsPMmxye6u1IqhG2MXlkz6EKRgx9rNf8D5a3rmdnPFOjTe-RLNYlV3rGbMx6g-YBQ9WX79GRV2YnhElyKXorzyV-gFJIhWNdopdb7U_U0kNxyAL7IW2b1U8W0oOo9qfQo7C1oIqv2CxZwjiAuA-C0vddV2yfy42SNTAX0Gho0eWQuTEwbegbIjaPjypV7g

$ curl 172.18.0.241/httpbin/get --header "Authorization: Bearer $TOKEN"
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "172.18.0.241",
    "User-Agent": "curl/7.81.0",
    "X-B3-Parentspanid": "197240bc0e5334f6",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "0ca2b1d079bfa951",
    "X-B3-Traceid": "d00e960c20fa0699197240bc0e5334f6",
    "X-Envoy-Attempt-Count": "1",
    "X-Envoy-Internal": "true",
    "X-Envoy-Original-Path": "/httpbin/get",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=22feb51ffec470cf6f2af2d8a910e2e3481fc149bcbe2d9f51b375a1105a5e40;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  },
  "origin": "10.244.1.1",
  "url": "http://172.18.0.241/get"
}

$ curl 172.18.0.241/httpbin/get -sS -o /dev/null -w "%{http_code}\n" --header "Authorization: Bearer $TOKEN"
200
```
### OutputClaimToHeaders, meneruskan claims ke header
Konfigurasi outputClaimToHeader 
https://istio.io/latest/docs/tasks/security/authentication/claim-to-header/

```
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  jwtRules:
  - issuer: "your-issuer"
    jwksUri: your-jwks-uri
    outputClaimToHeaders:
    - header: "x-sub"
      claim: "sub"
    - header: "x-iss"
      claim: "iss"
    - header: "x-userid"
      claim: "userid"
    - header: "x-company"
      claim: "company"
```
Apply konfigurasi outputClaimToHeaders

```
$ kubectl apply -f Kubernetes/istio-kind/yaml/jwt-authorization.yaml
```

Melihat header berupa claims JWT yang diteruskan istio RequestAuthentication

```
$ curl 172.18.0.241/httpbin/get --header "Authorization: Bearer $TOKEN"
{
  "args": {},
  "headers": {
    ...
    "X-Company": "VOBMcNBbMz",
    ...
    "X-Iss": "demo-api@issuer.cloud",
    "X-Sub": "jwt@example.com",
    "X-Userid": "ueOsPPSlHN"
  },
  ...
}
```
Sekarang httpbin microservice menerima *header* X-Company, X-Iss, X-Sub dan X-Userid yang merupakan claims dari token JWT yang diteruskan oleh istio authorization RequestAuthentication
