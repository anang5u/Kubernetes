# Istio - Authorization Policy dengan JWT

Sebelumnya telah dijelaskan secara singkat bagaimana cara install [istio](https://istio.io/latest/) di dalam sebuah cluster kubernetes, [Install Istio Gateway in Kubernetes Cluster](https://github.com/anang5u/Kubernetes/tree/master/istio/install-istio-in-kubernetes)

Sekarang kita akan mencoba bagaimana menerapkan *authorization policy* pada istio, yaitu dimana *enpoint* api hanya bisa diakses oleh *client* yang memiliki akses token JWT

Kita sudah memiliki [auth-microservice](https://github.com/anang5u/jwt-auth-microservice) yang akan digunakan sebagai service auth nya itu sendiri. Sebagai bahan pembelajaran, pada microservice auth tersebut disediakan endpoint */v1/login* dan */v1/jwks* yang nantinya akan digunakan di dalam konfigurasi *RequestAuthentication*

```
$ kubectl get pod
NAME        READY   STATUS    RESTARTS      AGE
service-a   2/2     Running   6 (22m ago)   3d23h
service-b   2/2     Running   6 (22m ago)   3d23h
```

Mencoba akses service tanpa token JWT
```
# Response
$ curl https://demo-api.anangsu13.cloud/app-a
<html>
  <body>
    <h1 style="color:green">Service A is online!</h1>
    <p>Service A is serving the traffic </p>
  </body>
</html>

# Status Code (http_code)
$ curl https://demo-api.anangsu13.cloud/app-a -sS -o /dev/null -w "%{http_code}\n"
200
```

## Deploy Auth Microservice
```
$ kubectl create deploy auth-microservice --image anangsu13/jwt-auth-microservice:v4.1
$ kubectl expose deploy auth-microservice --port 8080

$ kubectl get pod
NAME                                 READY   STATUS    RESTARTS      AGE
auth-microservice-56d68cc768-cdqfn   2/2     Running   0             65s
service-a                            2/2     Running   6 (35m ago)   4d
service-b                            2/2     Running   6 (35m ago)   4d

$ kubectl get svc
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
auth-microservice   ClusterIP   10.96.247.81   <none>        8080/TCP   2m26s
kubernetes          ClusterIP   10.96.0.1      <none>        443/TCP    4d1h
service-a           ClusterIP   10.96.61.14    <none>        80/TCP     4d
service-b           ClusterIP   10.96.53.123   <none>        80/TCP     4d
```

## Konfigurasi Auth Microservice
- Tambahkan `route auth/` pada konfigurasi *VirtualSevice* dengan `destination.host: auth-microservice`
  ```
  $ kubectl apply -f yaml/gateway.yaml
  gateway.networking.istio.io/istio-gateway unchanged
  virtualservice.networking.istio.io/istio-vs configured
  ```
- Apply konfigurasi `RequestAuthentication` dan `AuthorizationPolicy`
  ```
  $ kubectl apply -f yaml/jwt-auth.yaml
  requestauthentication.security.istio.io/jwt-on-ingress created
  authorizationpolicy.security.istio.io/require-jwt created
  ```
## Test Auth
- Request tanpa token JWT
  ```
  $ requestauthentication.security.istio.io/jwt-on-ingress created
  authorizationpolicy.security.istio.io/require-jwt created

  $ curl curl https://demo-api.anangsu13.cloud/app-a
  RBAC: access denied
  
  $ curl https://demo-api.anangsu13.cloud/app-a -sS -o /dev/null -w "%{http_code}\n"
  403
  ```
- Request token JWT
  ```
  $ curl -X POST https://demo-api.anangsu13.cloud/auth/v1/login
  {"token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjUwMCwiaXNzIjoieW91ci1pc3N1ZXIiLCJzdWIiOiJqd3RAZXhhbXBsZS5jb20ifQ.AxGijzMMTpdWY228M77ZpMzoWyaXa1fcKsVUgZ8cRQXX45--desAzcJcG8k6T_EIR1aJCZC0Wcw39So2Qz4vek7pKsiCQxIqiHuRgfffu4RuTNQp5a5f1nNL8lF_hc1juR218VmsueGmL1PkhfneCvCyGk7kSFvRZYaJOLF9ZZ-N2UbBLu52fs-GG6qdraoewU3cjzaIB3Sd_ViCRn-XlmCsQS2Hwy90-TXqvlH4YS_iLmGny1GvwnVBznWO0I1JUuD8To-wlSUleLtzoaG2GG_DtqHkAPkZ0hLlKg75TL-RHn1zHRo9HR2JMtj_tiXBGuKl6ALUsKNJ2qQm1bXF-g","user":{"id":"dummy-user-id"}}

  $ TOKEN=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjUwMCwiaXNzIjoieW91ci1pc3N1ZXIiLCJzdWIiOiJqd3RAZXhhbXBsZS5jb20ifQ.AxGijzMMTpdWY228M77ZpMzoWyaXa1fcKsVUgZ8cRQXX45--desAzcJcG8k6T_EIR1aJCZC0Wcw39So2Qz4vek7pKsiCQxIqiHuRgfffu4RuTNQp5a5f1nNL8lF_hc1juR218VmsueGmL1PkhfneCvCyGk7kSFvRZYaJOLF9ZZ-N2UbBLu52fs-GG6qdraoewU3cjzaIB3Sd_ViCRn-XlmCsQS2Hwy90-TXqvlH4YS_iLmGny1GvwnVBznWO0I1JUuD8To-wlSUleLtzoaG2GG_DtqHkAPkZ0hLlKg75TL-RHn1zHRo9HR2JMtj_tiXBGuKl6ALUsKNJ2qQm1bXF-g
  ```
- Request dengan token JWT
  ```
  $ curl  https://demo-api.anangsu13.cloud/app-a --header "Authorization: Bearer $TOKEN"
  <html>
  <body>
    <h1 style="color:green">Service A is online!</h1>
    <p>Service A is serving the traffic </p>
  </body>
  </html>

  $ curl  https://demo-api.anangsu13.cloud/app-a --header "Authorization: Bearer $TOKEN" -sS -o /dev/null -w "%{http_code}\n"
  200
  ```

  Selamat, sampai saat ini kita telah menerapkan *authorization* pada service di sebuah cluster kubernetes, dimana service hanya dapat diakses dari luar cluster jika memiliki token JWT yang valid.

  Selanjutnya, akan mencoba *EnvoyFilter* pada Istio yang berfungsi meneruskan *key-value* dari sebuah token dan akan meneruskannya ke sebuah *service* dalam bentuk *header* 

  Selengkapnya mengenai konfigurasi JWT pada Istio dapat mengacu pada dokumen resmi https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/