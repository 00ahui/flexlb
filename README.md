# FlexLB - Flexible Load Balancer

## About FlexLB

FlexLB is a flexible load balancer control plane to manage LB instances of Keepalived and HAProxy.
- <U>[flexlb-api](https://github.com/00ahui/flexlb-api)</U>: <I>control plane to manage LB instances</I>
- <U>[flexlb-client-go](https://github.com/00ahui/flexlb-client-go)</U>: <I>go client library to interact with flexlb-api</I>
- <U>[flexlb-kube-controler](https://github.com/00ahui/flexlb-kube-controller)</U>: <I>kubernetes controller to serve load balancer service</I>

## Concept

The purpose of FLexLB is to provide a <B>high reliable, scalable, and flexible</B> load balancer cluster, and provide a kubernetes controller to add external IP address for kubernetes load balancer type service. 
The following picture shows the concept of FlexLB. The FlexLB cluster normally contains two or more nodes (single node also suport), the LB nodes normally have three networks, mgmt network for control plane api calls, traffic network for application traffic, external network for ingress endpoint. The LB nodes have Keepalived and HAProxy installed, the flexlb-api server runs on each node and sync configs between each other through gossip protocol. The flexlb-api server starts a cluster endpoint listening on mgmt network virtual ip address, balance all the flexlb-api servers. The flexlb-kube-controller watches kubernetes service, and call flexlb-api to create LB instance for the service. The flexlb-api server add a virtual ip for each LB instance and start a haproxy process to forward the traffic from frontend external network to backend traffic network at L4 layer (tcp/udp), the backend traffic is forwarded to the node that the pods resides on, and then forwarded to pod ip through kubernetes service routing through iptables or ipvs.

![](https://raw.githubusercontent.com/00ahui/flexlb/main/images/flexlb-concept.png)

### FlexLB API Flow

The flexlb-api server can work in single node mode or cluster mode, when working in cluster mode, the flexlb-api servers sync LB instances config, status, and server status between each other through gossip protocol. A special LB instance is automatically created to serve flexlb cluster endpoint. Once the flexlb clients call flexlb-api cluster endpoint to create a LB instance, the request is forwarded to one of the flexlb-api server, the server create LB instance on local node, reload local keepalived, and sync the config to other nodes. Once other nodes receive the config,  they will create LB instance on it's local node and reload keepalived. The virtual interface of each node have random priority, the highest node will bring the virtual IP up. The flexlb-api watches instance virtual IP, once it finds it's brought up on local node, it starts haproxy instance. The LB instance update and delete operations have the same processing flow. The flexlb-api server's design is inspired by kubernetes declare api design, users only need to declare LB instance, the flexlb-api server watches and reconciles the LB instance, gurantee the final status of the LB instance to be right.

![](https://raw.githubusercontent.com/00ahui/flexlb/main/images/flexlb-api-flow.png)

### FlexLB Kube Controller Flow

The flexlb-kube-controller is a kubernetes controller to watch and feed external IP for kubernetes load balance type service. The controller supports multiple LB cluster, for each cluster support multiple IP pools. By declare a CRD called FlexLBCluster, users can define different external network interface and/or different IP ranges for each IP pool, they can set cluster and IP pool annotations for service to let the controller to allocate IP from the desired cluster and IP pool. Users can also declare a CRD called FlexLBInstance to let the controller create user customized LB instances. But for most case, users only need to declare load balancer type service, the controller automatically detect the endpointslice of the service, find the pod residential nodes, find the node traffic network and node port, create a FlexLBInstance and allocate external IP address for the service. 

![](https://raw.githubusercontent.com/00ahui/flexlb/main/images/flexlb-kube-controller-flow.png)

# Getting Start

## Run FlexLB API Server

### Prepare LB nodes

Typically prepare 3 LB nodes (linux) with the following network:
- mgmt network: used for node management and flexlb-api control plane listening address
- traffic network: used for application traffic and flexlb-api cluster gossip private network
- external network: to expose ingress endpoints

For example:

|          |   mgmt network  |  traffic network | external network |
|  :----:  |       :----:    |      :----:      |      :----:      |
|   node1  | 8.46.188.191/20 | 192.168.1.191/24 | 192.168.2.191/24 |
|   node2  | 8.46.188.192/20 | 192.168.1.192/24 | 192.168.2.192/24 |
|   node3  | 8.46.188.193/20 | 192.168.1.193/24 | 192.168.2.193/24 |
|  api-vip | 8.46.188.190/20 |                  |                  |
|   inst1  |                 |                  |  192.168.2.1/24  |

### Install Keepalived

Install and start keepalived on each LB nodes.

For example, install keepalived rpm binary on EulerOS 2.9:
```sh
rpm -ivh net-snmp-5.8-8.h6.eulerosv2r9.x86_64.rpm  net-snmp-libs-5.8-8.h6.eulerosv2r9.x86_64.rpm
rpm -ivh keepalived-2.0.20-16.h5.eulerosv2r9.x86_64.rpm
systemctl enable keepalived
systemctl start keepalived
```

### Intall HAProxy

Install HAProxy on each LB nodes (don't start).

For example, install haproxy rpm binary on EulerOS 2.9:
```sh
rpm -ivh haproxy-2.0.14-1.eulerosv2r9.x86_64.rpm
```

### Download FlexLB API Server

Download <U>[flexlb-api-0.4.0-linux-amd64.tar.gz](https://flexlb.gitee.io/releases/0.4.0/flexlb-api-0.4.0-linux-amd64.tar.gz)</U> and extract the tarball.

```sh
VERSION=0.4.0
wget https://flexlb.gitee.io/releases/${VERSION}/flexlb-api-${VERSION}-linux-amd64.tar.gz
mkdir flexlb-api
tar -zxf flexlb-api-${VERSION}-linux-amd64.tar.gz -C flexlb-api
```

### Generate self-signed certificate

Generate CA key and CA certs:
```sh
mkdir -p  /etc/flexlb/certs/
cd /etc/flexlb/certs/

DNS_NAME="example.com"

openssl genrsa -out ca.key 2048
openssl req -new -out ca.csr -key ca.key -subj "/CN=${DNS_NAME}"
openssl x509 -req -in ca.csr -out ca.crt -signkey ca.key -CAcreateserial -days 3650
```

Generate server key and certs:
```sh
openssl genrsa -out server.key 2048
openssl req -new -out server.csr -key server.key -subj "/CN=${DNS_NAME}"
openssl x509 -req -in server.csr -out server.crt -signkey server.key -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650
```

Generate client key and certs:
```sh
openssl genrsa -out client.key 2048
openssl req -new -out client.csr -key client.key -subj "/CN=${DNS_NAME}"
openssl x509 -req -in client.csr -out client.crt -signkey client.key -CA ca.crt -CAkey ca.key -CAcreateserial -days 3650
```

Copy to other nodes:
```sh
scp -r /etc/flexlb node2:/etc/flexlb
scp -r /etc/flexlb node3:/etc/flexlb
```

### Run FlexLB API Server

Edit flexlb-api-config.yaml on each node, and copy to /etc/flexlb/
```sh
vi flexlb-api-config.yaml
cp flexlb-api-config.yaml /etc/flexlb
```

Start FlexLB API Server on each node
```sh
chmod +x flexlb-api-${VERSION}-linux-amd64
./flexlb-api-${VERSION}-linux-amd64
```

### Test FlexLB API

Prepare 3 backend servers, for example, create 3 nginx server listening on traffic network:
```
192.168.1.141:30080
192.168.1.142:30080
192.168.1.143:30080
```

Prepare instance config for API post, for example: 
```
test/instance_template.json
```

Test instance create, list, modify, get, start, stop, delete
```sh
# get ready status
sh get_status.sh

# create instances
sh create_instance.sh inst1 192.168.2.1

# list instances
sh list_instances.sh
sh list_instances.sh inst1

# modify instance
# edit instance_template.json to add or remove backend servers
sh modify_instance.sh inst1 192.168.2.3

# show instance changes
sh get_instance.sh inst1

# stop & start instance
sh stop_instance.sh inst1
sh start_instance.sh inst1

# delete instances
sh delete_instance.sh inst1
```

## Run FlexLB CLI

### Download FlexLB CLI

Download <U>[flexlb-cli-0.4.0-linux-amd64.tar.gz](https://flexlb.gitee.io/releases/0.4.0/flexlb-cli-0.4.0-linux-amd64.tar.gz)</U> and extract the tarball.

```sh
VERSION=0.4.0
wget https://flexlb.gitee.io/releases/${VERSION}/flexlb-cli-${VERSION}-linux-amd64.tar.gz
tar -zxf flexlb-api-${VERSION}-linux-amd64.tar.gz -C flexlb-api
cd flexlb-api
chmod +x flexlb-api-${VERSION}-linux-amd64
```

### Run FlexLB CLI

Prepare intance config file:
```sh
TEMPLATE="test/instance_template.json"
NAME="inst1"
VIP="192.168.2.1"
sed "s/<NAME>/${NAME}/g; s/<VIP>/${VIP}/g" ${TEMPLATE} > test/inst1.json
```

Create instance:
```sh
./flexlb-api-${VERSION}-linux-amd64 -create test/inst1.json
```

List instance:
```sh
./flexlb-api-${VERSION}-linux-amd64 -list
./flexlb-api-${VERSION}-linux-amd64 -list -name inst1
```

Modify instance:
```sh
# edit /tmp/inst1.json
./flexlb-api-${VERSION}-linux-amd64 -modify /tmp/inst1.json
```

Get instance:
```sh
./flexlb-api-${VERSION}-linux-amd64 -get inst1
```

Stop/Start instance:
```sh
./flexlb-api-${VERSION}-linux-amd64 -stop inst1
./flexlb-api-${VERSION}-linux-amd64 -start inst1
```
Delete instance:
```sh
./flexlb-api-${VERSION}-linux-amd64 -delete inst1
```


## Run FlexLB Kube Controller


### Download Controller

Download <U>[flexlb-kube-controller-0.4.0.tar.gz](https://flexlb.gitee.io/releases/0.4.0/flexlb-kube-controller-0.4.0.tar.gz)</U> and extract the tarball.

```sh
VERSION=0.4.0
wget https://flexlb.gitee.io/releases/${VERSION}/flexlb-kube-controller-${VERSION}.tar.gz
mkdir flexlb-kube-controller
tar -zxf flexlb-kube-controller-${VERSION}.tar.gz -C flexlb-kube-controller
```

### Install Controller

```sh
# copy target kubernetes kubeconfig to ~/.kube/config
# run install.sh to config crd, rbac, controller, and flexlbcluster
cd flexlb-kube-controller
sh install.sh 8.46.188.190:8443 /etc/flexlb/certs/ca.crt /etc/flexlb/certs/client.crt /etc/flexlb/certs/client.key enp4s3 24 192.168.2.50 192.168.2.100 192.168.1.0/24
```

### Test Service

Create load balancer service, check the allocated external IP


