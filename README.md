# talos_docker_installation

#create a docker network to be shared between the registry and talos
docker network create  --driver "bridge" -o "com.docker.network.bridge.enable_icc"="true" -o "com.docker.network.bridge.name"="registry0"  --subnet=10.11.0.0/24 --gateway=10.11.0.1 --label "talos.owned"="true" --label "talos.cluster.name"="my-cluster" my-cluster

#create the registry and the mitm proxy by running
docker-compose -f docker-compose.mitmproxy.yml up -d

# pull etcd images and push them to the local registry
docker pull k8s.gcr.io/etcd:3.3.15-0
docker tag k8s.gcr.io/etcd:3.3.15-0 10.11.0.11:5000/etcd:3.3.15-0
docker push 10.11.0.11:5000/etcd:3.3.15-0
docker pull k8s.gcr.io/etcd:3.3.15-0
docker tag k8s.gcr.io/etcd:3.3.15-0 10.11.0.11:5000/k8s.gcr.io/etcd:3.3.15-0
docker push 10.11.0.11:5000/k8s.gcr.io/etcd:3.3.15-0

#create the talos cluster
talosctl --talosconfig talosconfig cluster create --name my-cluster --init-node-as-endpoint  --cidr 10.11.0.0/24 --input-dir talos_yaml
