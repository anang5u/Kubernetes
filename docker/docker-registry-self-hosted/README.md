# Private Docker Registry - Self Hosted

## Install Docker Engine
Jika *docker* belum tersedia, silahkan untuk mengikuti petunjuk installasi *docker engine* di https://docs.docker.com/engine/install/

```
$ docker -v
Docker version 25.0.3, build 4debf41
```

## Start the registry automatically
Docker registry akan dijalankan secara otomatis ketika docker *restart* `--restart=always` 
```
$ docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  registry:2

Unable to find image 'registry:2' locally
2: Pulling from library/registry
619be1103602: Pull complete
2ba4b87859f5: Pull complete
0da701e3b4d6: Pull complete
14a4d5d702c7: Pull complete
d1a4f6454cb2: Pull complete
Digest: sha256:f4e1b878d4bc40a1f65532d68c94dcfbab56aa8cba1f00e355a206e7f6cc9111
Status: Downloaded newer image for registry:2
4a11f5b570b0796173301bff6657c6d556f7e7e2853e2c25eb9ecb35e0b99593
```

## Test Docker Registry
### Create Dockerfile
```
FROM debian:latest
CMD  tail -f /dev/null
```
### Build docker
```
$ docker build -t localhost:5000/my-debian:v1 .  
```

### Push image
```
$ docker push localhost:5000/my-debian:v1
The push refers to repository [localhost:5000/my-debian]
1a5fc1184c48: Pushed
v1: digest: sha256:94b7af0f52307e84c65d296883b84ce0d7a25658f49e3fd73ee7765c66fc1804 size: 528

# Cek docker images
$ docker images localhost:5000/my-debian
REPOSITORY                 TAG       IMAGE ID       CREATED       SIZE
localhost:5000/my-debian   v1        bb2f9f68e082   3 weeks ago   117MB
```
### Pull image dari local registry
```
$ docker pull localhost:5000/my-debian:v1
v1: Pulling from my-debian
7bb465c29149: Already exists
Digest: sha256:94b7af0f52307e84c65d296883b84ce0d7a25658f49e3fd73ee7765c66fc1804
Status: Downloaded newer image for localhost:5000/my-debian:v1
localhost:5000/my-debian:v1
```

### Remove image
```
$ docker image remove localhost:5000/my-debian:v1
```

## Stop dan remove local registry
```
$ docker container stop registry
$ docker container stop registry && docker container rm -v registry
```
Lebih lengkap, bisa mengikuti dokumentasi resmi https://distribution.github.io/distribution/about/deploying/
