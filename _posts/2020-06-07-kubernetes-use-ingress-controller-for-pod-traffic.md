---
layout: post
title:  "Use ingress controller to route traffic to kubernetes pods"
author: Prabhu
categories: [ Kubernetes, Azure ]
image: assets/images/kubernetes.jpg
featured: true
---

Ingress controllers are used to route traffic to kubernetes services which exposes the applications running on the pods. Set of rules called Ingress is used to route traffic from outside the cluster to services in the cluster.


There are lot of ingress controllers avaiable for us. Few of them are
* AKS Application Gateway ingress controller
* AWS ALB ingress controller
* Nginx ingress controller
* Gloo
etc,.

Now let us use Nginx ingress controller to route traffic to AKS cluster services


## Architecture

In this architecture, Nginx controller is deployed on a sperate namespace and we have created other namespaces to segregate applications. Individual Ingress is deployed to the respective namespaces. Nginx controller uses the Ingress rules and route traffic to the services accordingly.

![Ingress controller](assets/images/2020-06-07/img1.jpg)


## Deployment

Lets deploy Nginx controller to a separate namespace.

![Nginx controller](/assets/images/2020-06-07/img2.jpg)

For this deployment, related resources such as Pods, service, deployment and replicaset are cleated.

![Nginx resources]({{site.url}}/assets/images/2020-06-07/img3.jpg)

Now lets create a dev namespace and deploy two sample pods and service, aks-helloworld and nginx.

![aks-helloworld]({{site.url}}/assets/images/2020-06-07/img4.jpg)

![nginx]({{site.url}}/assets/images/2020-06-07/img5.jpg)

Following resources would be created in dev namespace as part of the deployment.

![app resources]({{site.url}}/assets/images/2020-06-07/img6.jpg)

## Ingress

Now let us deploy the Ingress rule. Remember this has to be deployed to the same namespace as the services, i.e dev namespace.

![Ingress]({{site.url}}/assets/images/2020-06-07/img7.jpg)

## Testing

Now, for us to test the ingress controller, we have to use the Public IP address that is created as part of the service/ingress-nginx in ingress-nginx namespace.

Use the path that is set in ingress rule to access the respective services.

![aks-helloworld]({{site.url}}/assets/images/2020-06-07/img8.jpg)

![nginx]({{site.url}}/assets/images/2020-06-07/img9.jpg)


#### Using the same method you can have multiple namespaces created and deploy ingress rules to the respective namespaces. !!! Cheers !!!