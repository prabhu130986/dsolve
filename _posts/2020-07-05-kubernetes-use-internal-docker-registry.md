---
layout: post
title:  "Use internal docker registry to pull image to AKS"
author: Prabhu
categories: [ Kubernetes, Azure, Docker ]
image: assets/images/AKD.jpg
featured: true
---

All large organisations will try to host workloads in an atonomous / private environment even though they host in public cloud providers by blocking public exposure of their systems. 

Now Azure gives us option to provision private AKS cluster thus restricting the traffic between control plane and the node pool remains within the private network. 

In-order to deploy services / pods to the cluster we can pull the container images from Azure container registry or any other public registries like docker.io. But how can we restrict this public exposure and maintain a private registry and make AKS to pull from the private network.

Let's do it. I've used the below setup to demonstrate this.

![Architecture](/assets/images/2020-07-05/img1.JPG)

I have the below resource created

* AKS cluster
* A linux VM with docker. "registry" container is pulled and running in it
* Private DNS zone to register the hosts

## Azure resources creation

Create the following resources in Azure via portal or cli or script based on your convenient. 

* Create a resource group and a virtual network
* Create a private DNS zone. Add the virtual network created in "Virtual network links" option and enable Auto-registration. I have created "dsolve.cloud.intranet"
* Create a linux VM (I've used RHEL 7) and attach to the virtual network. I have created with name "docker-registry"
* Create a AKS cluster and attach to the same virtual network

Since we have enabled Auto-registration in the DNS zone, VMs that are created in the virtual network will be registered to DNS. You can see below that VM and AKS node is registered in the DNS.

![DNS Zone](/assets/images/2020-07-05/img2.JPG)

## Docker registry setup

We are going to use the VM as internal container registry. For that we are going to use "registry" container and have it running as a docker container.

Install docker using the below steps

```
[root@docker-registry ~]# sudo yum install -y yum-utils

[root@docker-registry ~]# sudo yum-config-manager \
>     --add-repo \
>     https://download.docker.com/linux/centos/docker-ce.repo

[root@docker-registry ~]# sudo yum install -y http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.107-3.el7.noarch.rpm

[root@docker-registry ~]# sudo yum install -y docker-ce docker-ce-cli containerd.io

[root@docker-registry ~]# service docker start
```

Now that docker is installed, let us create a self-signed certificate. This is needed by the cilents, either AKS cluster or any other system in the intranet to connect to registry and pull / push image.

Since we have created a private DNS zone, let us use the DNS name of the VM and use it as the CN to create self-signed certificate. In this case, my DNS name will be docker-registry.dsolve.cloud.intranet

```
[root@docker-registry ~]# mkdir -p /root/registry/certs

[root@docker-registry ~]# openssl req \
>   -newkey rsa:4096 -nodes -sha256 -keyout /root/registry/certs/domain.key \
>   -x509 -days 365 -out /root/registry/certs/domain.crt \
>   -subj "/C=SG/ST=SG/L=SG/O=SG/CN=docker-registry.dsolve.cloud.intranet"

```

Let us start the docker container using "registry:2" image

```
[root@docker-registry ~]# docker run -d \
>   --restart=always \
>   --name registry \
>   -v /root/registry/certs:/certs \
>   -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
>   -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
>   -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
>   -p 443:443 \
>   registry:2
```

![docker registry](/assets/images/2020-07-05/img3.JPG)

Now that the docker container is running, let us use the self-signed certificate to push an image to registry. For that we have to make the client (in this case, it is the same VM) to trust the certificate. Follow the below steps to do that.

```
[root@docker-registry ~]# mkdir -p /etc/docker/certs.d/docker-registry.dsolve.cloud.intranet

[root@docker-registry ~]# cat /root/registry/certs/domain.crt > /etc/docker/certs.d/docker-registry.dsolve.cloud.intranet/ca.crt
```

CA certificate should be placed under certs.d folder under the DNS have which we are going to connect. Since this is a self-signed certificate we copy the domain.crt file as ca.crt to the folder.

Now we can test if we are able to connect to the registry.

```
[root@docker-registry ~]# docker pull nginx

[root@docker-registry ~]# docker tag nginx docker-registry.dsolve.cloud.intranet/nginx
```

![docker push](/assets/images/2020-07-05/img4.JPG)

We can see that the image is successfully pushed to the registry. Now let us move to the AKS setup.

## AKS Setup

AKS cluster is also a client which need to connect to the registry and pull image. But how do we add the trust cert to the cluster???

For that we are going to create Secret and DaemonSet to the kube-system of the cluster. So this will replicate the certificate to all existing as well as future nodes of the cluster when scaled-up.

Get the base64 encoded content of the rootCA certificate using the below command. Since this is a self-signed we can encode the domain.crt that we created earlier.

![encode](/assets/images/2020-07-05/img5.JPG)

Let us create a yaml manifest to create secret with the above encoded value

```
apiVersion: v1
kind: Secret
metadata:
  name: registry-ca
  namespace: kube-system
type: Opaque
data:
  registry-ca: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZxekNDQTVPZ0F3SUJBZ0lKQU1WV0N5aEptcGxETUEwR0NTcUdTSWIzRFFFQkN3
```

Create another yaml manifest for DaemonSet.

```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: registry-ca
  namespace: kube-system
  labels:
    k8s-app: registry-ca
spec:
  template:
    metadata:
      labels:
        name: registry-ca
    spec:
      containers:
      - name: registry-ca
        image: busybox
        command: [ 'sh' ]
        args: [ '-c', 'cp /home/core/registry-ca /etc/docker/certs.d/docker-registry.dsolve.cloud.intranet/ca.crt && exec tail -f /dev/null' ]
        volumeMounts:
        - name: etc-docker
          mountPath: /etc/docker/certs.d/docker-registry.dsolve.cloud.intranet
        - name: ca-cert
          mountPath: /home/core
      terminationGracePeriodSeconds: 30
      volumes:
      - name: etc-docker
        hostPath:
          path: /etc/docker/certs.d/docker-registry.dsolve.cloud.intranet
      - name: ca-cert
        secret:
          secretName: registry-ca
```

Retrieve the kubeconfig of the cluster using below command.

```
az aks get-credentials --resource-group rg-aks --name myaks -a
```

Let us apply the yaml manifests to the cluster.

```
[root@docker-registry certs]# kubectl apply -f registry-secret.yml

[root@docker-registry certs]# kubectl apply -f registry-ca.yml
```

Now let us create a yaml manifest to deploy nginx deployment using the image which we earlier pushed to the docker registry

```
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
  labels:
    app: nginx

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: docker-registry.dsolve.cloud.intranet/nginx
          ports:
            - name: http-port
              containerPort: 8080

```

Now apply this as below.

```
[root@docker-registry certs]# kubectl apply -f nginx.yml
```

![RUNNING](/assets/images/2020-07-05/img6.JPG)

We can see the pod is running. Let us describe the pod and see details.

```
kubectl describe pod/nginx-5cc77f48f6-5zhdp -n nginx
```

![POD](/assets/images/2020-07-05/img7.JPG)

We can see that AKS cluster successfully pulled the image from the internal docker registry.

#### This is a real time use case used by many organisations. I have used private DNS and self-signed cert to demonstrate this. Hope this helps !!! Cheers !!!