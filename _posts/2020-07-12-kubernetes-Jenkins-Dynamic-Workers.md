---
layout: post
title:  "Dynamic Jenkins worker setup with AKS"
author: Prabhu
categories: [ Kubernetes, Azure, Jenkins ]
image: assets/images/AKDJ.jpg
featured: true
---

Jenkins is a famous CI CD tool used by many organisations which works on master-worker concept. That means, Master node holds the configurations and administer the jobs and worker nodes execute the jobs. 
The conventional way for this setup is done by installing jenkins agents on multiple servers and attach them to the master node. This means, when there are no jobs to execute the agents will be idle and un-used.

In this era of on-demand and dymanic resources, let us use Azure Kubernetes Service to do this Jenkins setup. Below is the architecture we are going to setup.

![Architecture](/assets/images/2020-07-12/img1.JPG)

Here we will have one pod for jenkins master always up and running. All worker pods will be created only when a job is triggered. Thus saving resources and cost.

Let us use the AKS, docker and internal registry setup we already did [here]({{ site.url }}{% link _posts/2020-07-05-kubernetes-use-internal-docker-registry.md %}).

Along with the existing setup we need below additional resources.

* Docker image for Jenkins master
* Docker image for Jenkins agent

## Push docker images to internal registry

I already have a docker image for jenkins master and jenkins agent in my docker hub registry. I have also installed ansible on the jenkins agent image for illustration.

I'm going to pull the images to the docker VM.

![docker pull master](/assets/images/2020-07-12/img2.JPG)
![docker pull agent](/assets/images/2020-07-12/img3.JPG)

Now i'm chainging the tag and pushing the image to the internal registry docker container that is running.

![docker tag](/assets/images/2020-07-12/img4.JPG)
![docker push master](/assets/images/2020-07-12/img5.JPG)
![docker push agent](/assets/images/2020-07-12/img6.JPG)

## Deploy Jenkins Master to AKS cluster

Now that the images are pushed to the registry, let us deploy the Jenkins Master pod to the AKS cluster. Below is the manifest file which i used.

```
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
  labels:
    app: jenkins

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: docker-registry.dsolve.cloud.intranet/jenkins-master
          env:
            - name: JAVA_OPTS
              value: -Djenkins.install.runSetupWizard=false
          ports:
            - name: http-port
              containerPort: 8080
            - name: jnlp-port
              containerPort: 50000
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp1
  namespace: jenkins
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: 50000
  selector:
    app: jenkins

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
  type: LoadBalancer
  selector:
    app: jenkins
```

I'm creating a namespace jenkins so as to segregate the resources and container image is pointed to the internal registry. Let us apply this manifest.

![kubectl apply](/assets/images/2020-07-12/img7.JPG)

We can notice that image is pulled from internal registry.

![master pull](/assets/images/2020-07-12/img8.JPG)

We can see that the resources are created and jenkins master pod is created and running.

![master pod](/assets/images/2020-07-12/img9.JPG)

Note down the IP address for "ClusterIP" Service Type which will be needed during Jenkins configuration later.

## Jenkins Configuration 

Now that the jenkins mater pod is running, we can access the jenkins console using the external IP address which you can see under LoadBalancer.

![jenkins console](/assets/images/2020-07-12/img10.JPG)

To configure Jenkins for AKS "Kubernetes" plugin must be installed. I have already installed and enabled this plugin in the docker image, so no need to install it separately. You can see it installed under "Manage Jenkins" --> "Manage Plugins"

![plugin](/assets/images/2020-07-12/img11.JPG)

Now navigate to "Manage Jenkins" --> "Configure System"

![configure](/assets/images/2020-07-12/img12.JPG)

Scroll to "Cloud" section, click "Add a new cloud" and select "Kubernetes". 

![cloud](/assets/images/2020-07-12/img13.JPG)

Scroll to "Credentials" and click "Add" and select "Jenkins". Select options as the following and add kubeconfig file (Remember we pulled the config using get-credentials) and click "Add".

![credential](/assets/images/2020-07-12/img14.JPG)

Now select the kubeconfig from drop down and click "Test Connection". You will see "Connection test successful" if jenkins is able to connect to the AKS cluster, else you will get an error if kubeconfig is not working.

![connection](/assets/images/2020-07-12/img15.JPG)

In "Jenkins tunnel" section, fill in the ClusterIP address you have noted above and the port number which is 50000.

![tunnel](/assets/images/2020-07-12/img16.JPG)

Scroll down and click on "Add Pod Template". Fill the details as below.

* Name: jnlp-ansible
* Namespace: jenkins
* Labels: jnlp-ansible

![pod](/assets/images/2020-07-12/img17.JPG)

Click on "Add Container" and "Container Template". Fill the details as below.

* Name: jnlp
* Docker image: docker-registry.dsolve.cloud.intranet/jnlp-ansible

Empty values of the below options.

* Command to run	
* Arguments to pass to the command	

![template](/assets/images/2020-07-12/img18.JPG)

And click "Save"

## Create Job

Let us create a job with below options.

Click "New item" --> "Freestyle Project". In "Label Expression" type "jnlp-ansible". This is the label we created earlier.

![job](/assets/images/2020-07-12/img19.JPG)

Scroll to build section, select "Execute Shell" and provide some commands to execute. Click Save.

![job1](/assets/images/2020-07-12/img20.JPG)

Create few more jobs similar to the one which is created now. I have created 3 similar jobs.

## Dynamic workers in action

Now, execute all the jobs at-once. We can see that 3 workers created in "Build Executor Status" section.

![workers](/assets/images/2020-07-12/img21.JPG)

In the meantime, we can see 3 pods created in the jenkins namespace which corresponds to each worker which is created.

![pods](/assets/images/2020-07-12/img22.JPG)

Once the job is completed, agent pods will get destroyed.

#### This setup can be expanded to next level using AKS cluster auto-scaling. If there are more jobs triggered at same time and if the existing node is out of capacity, new node will be added to the AKS cluster and new pods will be attached to it and cluster will shrink back to its minimum limit, thus saving cost and resources !!! Cheers !!!