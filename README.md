# talos_docker_installation 
Following this instruction will create a local talos cluster using docker. 

## Prerequisites
I did this in a 18.04 Ubuntu VM with 
- docker installed and configured
- openssl
- talosctl v0.4.1


### create a docker network to be shared between the registry and talos
We will create a docker bridge network 10.11.0.0/24 that will be used by both talos and the registry. This is to ensure that talos has access to the registry.
Note that in order for talos to reuse existing network, two conditions should be met:
- The network should have the same name as the talos cluster
- The network should have the following labels: `talos.cluster.name"="<cluster name>"` and `"talos.owned"="true"`


```
docker network create  --driver "bridge" -o "com.docker.network.bridge.enable_icc"="true" -o "com.docker.network.bridge.name"="registry0"  --subnet=10.11.0.0/24 --gateway=10.11.0.1 --label "talos.owned"="true" --label "talos.cluster.name"="my-cluster" my-cluster
```

### create the registry and the mitm proxy by running
We will use the docker compose file *docker-compose.registry_mitmproxy.yml* to deploy the private docker registry and the mitmproxy.
Note that the registry uses self signed certificates and in order to make talos able to pull the images we needed to add the domain.crt to talos at */etc/ssl/certs/ca-certificates*
The certificates for the registry were created using the following commands and then saved to the *certs* directory that is mounted in the registry image */certs* directory.

```
openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key  -x509 -days 365 -out domain.crt
```

the ip address of the registry 10.11.0.11 was defined in *subjectAltName=IP:10.11.0.11* in the file */etc/ssl/openssl.cnf*

```
docker-compose -f docker-compose.registry_mitmproxy.yml up -d
```

### pull etcd images and push them to the local registry
```
docker pull k8s.gcr.io/etcd:3.3.15-0
docker tag k8s.gcr.io/etcd:3.3.15-0 10.11.0.11:5000/etcd:3.3.15-0
docker push 10.11.0.11:5000/etcd:3.3.15-0
docker pull k8s.gcr.io/etcd:3.3.15-0
docker tag k8s.gcr.io/etcd:3.3.15-0 10.11.0.11:5000/k8s.gcr.io/etcd:3.3.15-0
docker push 10.11.0.11:5000/k8s.gcr.io/etcd:3.3.15-0
```

### create the talos cluster
```
talosctl --talosconfig talosconfig cluster create --name my-cluster --init-node-as-endpoint  --cidr 10.11.0.0/24 --input-dir talos_yaml
```
