# exo-ornikar



## Getting started

Nodejs App and PHP App From Docker To Kubernetes Cluster with traefik and metallb.

## Provisionning of a VM

* Install of Minikube

The package is in the the script shell bootstrap.

In the script bootstrap.sh that I had explain in this part [Click on the link to see more details](https://github.com/sory89/Vagrant-Machine-Ornikar/edit/main/README.md?). 

Requirements :

I have install kubectl, Here I use Docker as a provider :)

To see all the command click on [bootstrap.sh](https://github.com/sory89/Vagrant-Machine-Ornikar/blob/main/bootstrap.sh?)

to start kubernetes :

```
$ minikube start --force --driver=docker

and Check if the service is running

root@server00:~/MiniKube-K8S/ornikar-vagrant-code/Vagrant-Machine-Ornikar# minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

```

After provisionnning, if you have a errorr.
delete the minikube docker with the follow command and delete the log file

```
$ minikube delete

$ rm -rf /tmp/juju* && rm-rf /tmp/mini*

and restart the minikube with

$ minikube start --force --driver=docker
```


* Install of matallb

Metallb is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.

More see the [link](https://metallb.universe.tf/?)

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

to see the IP adress of LoadBalancer

```
root@server00:~/MiniKube-K8S/ornikar-vagrant-code/Vagrant-Machine-Ornikar# minikube ip
192.168.49.2
```

MetalLB is instructed to hand out addresses from 192.168.49.1 to 192.168.49.5. After that, we’ll create a config map in the metallb-system namespace.

```
root@server00:~/kubernetes/exo-ornikar# cat config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.49.1-192.168.49.5
```

than

```
kubectl create -f config.yaml
```

```
root@server00:~/kubernetes/exo-ornikar# kubectl -n metallb-system get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-6b78bff7d9-6wgb8   1/1     Running   2          25h
pod/speaker-nhnfd                 1/1     Running   1          25h

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   1         1         1       1            1           kubernetes.io/os=linux   25h

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           25h

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-6b78bff7d9   1         1         1       25h
```

I deploy the nginx to test the loadbalancer

```
kubectl create deploy nginx --image nginx


kubectl get all
kubectl expose deploy nginx --port 80 --type LoadBalancer

kubectl get svc
root@server00:~/kubernetes/exo-ornikar# kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP        26h
nginx        LoadBalancer   10.100.35.154   192.168.49.2   80:32201/TCP   25h
nodejs-app   ClusterIP      10.102.42.24    <none>         80/TCP         24h
php-app      ClusterIP      10.110.51.218   <none>         80/TCP         17h

lynx 192.168.49.2

```


* Install of traefik

to install [traefik](https://doc.traefik.io/traefik/getting-started/install-traefik/?), I use [HELM](https://helm.sh/?). 

```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm repo list
helm search repo traefik
helm show values traefik/traefik > /tmp/traefik-values.yaml

```


```
vi /tmp/traefik-values.yaml

Then edit traefik-values.yaml and change the 
persistence:
  enabled: false
  
  by
  
persistence:
  enabled: true
  
helm install traefik traefik/traefik --values /tmp/traefik-values.yaml -n traefik --create-namespace
```

Check the namespace 
```
root@server00:~/kubernetes/exo-ornikar# kubectl get ns
NAME              STATUS   AGE
default           Active   26h
kube-node-lease   Active   26h
kube-public       Active   26h
kube-system       Active   26h
metallb-system    Active   25h
traefik           Active   25h
```

Run the traefik
```
root@server00:~/kubernetes/exo-ornikar# kubectl -n traefik get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/traefik-f4f8dfb5b-hrv4b   1/1     Running   1          26h

NAME              TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/traefik   LoadBalancer   10.96.137.163   192.168.49.1   80:30616/TCP,443:30062/TCP   26h

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           26h

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-f4f8dfb5b   1         1         1       26h
```

and 

```
root@server00:~/kubernetes/exo-ornikar# kubectl -n traefik port-forward traefik-f4f8dfb5b-hrv4b 9000:9000
Forwarding from 127.0.0.1:9000 -> 9000
Forwarding from [::1]:9000 -> 9000
.
.
.
```

to check the dashboard "localhost:9000/dashboard/"


## Deploy An App in the Docker and Kubernetes

### Node App in Docker

* [Docker](https://www.docker.com/?) and [Kubernetes](https://www.Kubernetes.com/?)

In Dockerfile

```
FROM node:13
WORKDIR /app
COPY package.json /app
RUN npm install
COPY . /app
CMD node src/index.js
EXPOSE 3000
```

1* This download the image node version 13 from Dockerhub

2* WORKDIR, tells docker the working directory of our image (in our case it is /app), it cd on lunix. 

2* CMD or RUN commands execute in this folder

3* CP stands for copy; Here, we’re copying package.json file to /app

4* RUN : launch the command

5* CMD : Execute the command

6* EXPOSE 3000, here it informs the user container (using this image) that it needs to open port 3000.


build a image with this command and create container

```
docker build -t node-server .

docker run -d --name node-app -p 3000:3000 node-server

```

## Registry Docker Hub and GitLab

First connect to Docker hub registry with your login or GitLab registry
```
docker tag node-server sorydiallo89/nodejs-starter

docker push sorydiallo89/nodejs-starter

```

### PHP App in Docker

In Dockerfile

```
FROM php:7.2-cli
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git
WORKDIR /app
RUN curl -sS https://getcomposer.org/installer | \
    php -- --install-dir=/usr/bin/ --filename=composer
COPY composer.json /app
RUN composer install --no-scripts --no-autoloader
COPY . /app
CMD php public/index.php
EXPOSE 3000
```

I do the same :)


## YAML File To Create A Deployment In Kubernetes Cluster

YAML is a human-readable extensible markup language. It’s used in Kubernetes to create an object in a declarative way.

The deploy a Docker image node in kubernetes

This file is a deployment nodejs-deploy.yml

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nodejs-app
  namespace: default
  labels:
    app: nodejs-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-app
        image: "sorydiallo89/nodejs-starter"
        ports:
          - containerPort: 3000
```

and service nodejs-svc.yml

```
apiVersion: v1
kind: Service
metadata:
  name: php-app
  namespace: default
spec:
  selector:
    app: php-app
  ports:
  - name: http
    targetPort: 3000
    port: 80
```


The deploy a Docker image PHP in kubernetes

This file is a deployment php-deploy.yml

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: php-app
  namespace: default
  labels:
    app: php-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-app
  template:
    metadata:
      labels:
        app: php-app
    spec:
      containers:
      - name: php-app
        image: "sorydiallo89/php-starter"
        ports:
          - containerPort: 3000
```

and service nodejs-svc.yml

```
apiVersion: v1
kind: Service
metadata:
  name: php-app
  namespace: default
spec:
  selector:
    app: php-app
  ports:
  - name: http
    targetPort: 3000
    port: 80
```

then the traefik-ingress.yml

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: ornikar.dev
    http:
      paths:
      - backend:
          service:
            name: php-app
            port:
              number: 80
        path: /world
        pathType: Prefix
  - host: ornikar.dev
    http:
      paths:
      - backend:
          service:
            name: nodejs-app
            port:
              number: 80
        path: /hello
        pathType: Prefix
```

To create deploy, service, ingress

```
kubectl create -f .
```

then : 

http://ornikar.dev/hello and http://ornikar.dev/world
