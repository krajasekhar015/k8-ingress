# Ingress Controller

An Ingress Controller is a tool in Kubernetes that manages how external traffic (like web requests) gets into your cluster and reaches the right application inside the cluster. It acts like a gatekeeper, directing incoming requests to the correct service based on rules you set.

Kubernetes cluster is about cluster plane and worker-nodes.

In kubernetes cluster, loadbalancer is purely external component

Bydefault, in general kubernetes EKS creates classic loadbalancer
    - This is old generation which has some limitations. It is not intelligent
    - It cannot route traffic to different target groups
    - It is not recommended by AWS

Whereas Application Load Balancer (ALB) is intelligent
    - It routes traffic to multiple target groups based on host or context rules
    - It works on Layer 7 and it is preferred by AWS

Why we need Ingress Controller?

- If we want give interent access to our application running in K8, then we have to provision ingress controller. We are using ALB as our ingress controller. We installed aws load balancer controller drivers through helm charts and given appropriate permissions.

- we create ingress resource/object for the apps that requires external access

> By opening nodeport we will get internet access. But opening nodeport is not secure

- Suppose there is HDFC bank with different applications like
    - netbanking.hdfc.com --> It will route traffic to netbanking
    - corporatebanking.hdfc.com --> It will route traffic to corporatebanking
    - HDFC bank will have k8 cluster where all the applications will run


**How you setup Ingress controller and what is ingress controller? (Interview Question)**
- The purpose of Ingress controller is to provide internet access to our applications running in kubernetes
- We are using EKS cluster and ALB as our ingress controller
- As part of the setup, we provided the oidc provider and created AWS policies
- We installed AWS LoadBalancerController drivers through helm charts and given appropriate permissions
- We create ingress resource/object for the apps that requires external access

> Create IngressController Using the commands mentioned in readme.MD file

After installing all the commands, run the following command to see the installed drivers
```
kubectl get pods -n kube-system
```
- Here, we can see aws-load-balancer-controller drivers. These are responsible to connect EKS cluster and loadbalancer


- Now, we will write `ingress.yaml`

- We will write two applications
    - If we give https://app1.krajasekhar015.online then it should go to app1
    - If we give https://app2.krajasekhar015.online then it should go to app2

- Create app1 folder and inside the folder create Dockerfile

```
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/index.html
RUN echo "<h1>Hello, I am from APP-1</h1>" > /usr/share/nginx/html/index.html
```

Apply the following commands:
```
docker build -t krajasekhar015/app1:v1 . 
```
```
docker login -u krajasekhar015
```
``` 
docker push krajasekhar015/app1:v1
```

- Now, we need to write `manifest.yaml` file 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  labels: # these are replicaset labels
    name: app1
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 1
  selector:
    # these are used to select the pod to create replicas
    matchLabels:
      name: app1
      tier: frontend
  # this is pod definition
  template:
    metadata:
      # these labels belongs to pod
      labels:
        name: app1
        tier: frontend
    spec:
      containers:
      - name: app1
        image: mahalakshmi2997/app1:v1
---
kind: Service
apiVersion: v1
metadata:
  name: app1
spec:
  selector:
    name: app1
    tier: frontend
  ports:
  - name: nginx-svc-port
    protocol: TCP
    port: 80 # service port
    targetPort: 80 # container port
```

```
kubectl apply -f manifest.yaml
```
``` 
kubectl get pods 
```
- Here, service type is ClusterIP

- We have created IngressController which is having dedicated resource called `ingress resource`
- Here, We will write routing rules
- For this purpose, we need to create certicate.
    - Go to AWS -> certificate Manager -> click on request 
    - crequest for public certificate 
    - fully qualified domain name -> *.krajasekhar015.online
    - click on request 

- Now we need to create route53 records
    - click on create records in Route53

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  labels: # these are replicaset labels
    name: app1
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 1
  selector:
    # these are used to select the pod to create replicas
    matchLabels:
      name: app1
      tier: frontend
  # this is pod definition
  template:
    metadata:
      # these labels belongs to pod
      labels:
        name: app1
        tier: frontend
    spec:
      containers:
      - name: app1
        image: joindevops/app1:v1
---
kind: Service
apiVersion: v1
metadata:
  name: app1
spec:
  selector:
    name: app1
    tier: frontend
  ports:
  - name: nginx-svc-port
    protocol: TCP
    port: 80 # service port
    targetPort: 80 # container port
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: app1
    annotations:
      #kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:315069654700:certificate/eefec79d-8012-4e7e-a530-baf154b833f2
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
      alb.ingress.kubernetes.io/tags: Environment=dev,Team=test
      alb.ingress.kubernetes.io/group.name: expense
spec:
    ingressClassName: alb
    rules:
    - host: "app1.daws81s.online"
      http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: app1
              port:
                number: 80
```

- In starting of the kubernetes sessisons, we have learnt about lables and annotations
- Lables are selectors used for selecting the internel resources of kubernetes (Like service need to attched to pod, deployment needs to select pod)
- Annotations are used to select externel resources (Here there is no limit for special characters, character length)

- If we see in the above code last section, there is annotations field.
- This ingress will create application load balancer
    - Internet Facing
    - IP Based
    - Certificate ARN
    - Listerner ports
    - Tags
    - Group
- Here, grouping is very important because if we don't mention group then it each ingress will create separate load-balancer

```
kubectl apply -f manifest.yaml 
```
```
kubectl get ingress 
```
- we will get the load balancer DNS value
```
kubectl get pods -o wide
```

![alt text](img/ingress.svg)

- Now, we need to create Route53 Record
    - Record name -> app1.krajasekhar015.online 
    - Alias 
	      -> Route traffic to -> Alias to application and classic load balancer 
	  - Region -> Us-east(N.virginia)
	  - Select the load balancer DNS value 
    - click on create
  
- Check the application on browser with this URL `https://app1.krajasekhar015.online`


- Now, create app2 folder and inside the folder create Dockerfile

```
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/index.html
RUN echo "<h1>Hello, I am from APP-2</h1>" > /usr/share/nginx/html/index.html
```

Apply the following commands:
```
docker build -t krajasekhar015/app2:v1 . 
```
```
docker login -u krajasekhar015
```
``` 
docker push krajasekhar015/app2:v1
```

- Now, we need to write `manifest.yaml` file 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  labels: # these are replicaset labels
    name: app2
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 1
  selector:
    # these are used to select the pod to create replicas
    matchLabels:
      name: app2
      tier: frontend
  # this is pod definition
  template:
    metadata:
      # these labels belongs to pod
      labels:
        name: app2
        tier: frontend
    spec:
      containers:
      - name: app2
        image: mahalakshmi2997/app2:v1
---
kind: Service
apiVersion: v1
metadata:
  name: app2
spec:
  selector:
    name: app2
    tier: frontend
  ports:
  - name: nginx-svc-port
    protocol: TCP
    port: 80 # service port
    targetPort: 80 # container port
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: app2
    annotations:
      kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:315069654700:certificate/eefec79d-8012-4e7e-a530-baf154b833f2
      alb.ingress.kubernetes.io/listen-ports: '[ {"HTTPS": 443}]'
      alb.ingress.kubernetes.io/tags: Environment=dev,Team=test
      alb.ingress.kubernetes.io/group.name: expense
spec:
    rules:
    - host: "app2.krajasekhar015.online"
      http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: app2
              port:
                number: 80
```

```
kubectl apply -f manifest.yaml 
```
```
kubectl get ingress 
```
- we will get the load balancer DNS value
```
kubectl get pods -o wide
```

**Ingress Controller**
- Ingress controller is used to provide external access to the applications running inside k8. In EKS we can use ALB as ingress controller 
- We install aws load balancer controller to connect with ALB and provide permissions to EKS 
- We have a resource called ingress to create ALB, Listeners, rules and target groups

**Difference between labels vs annotations**
**labels** 
- It have limited length and special characters are not allowed.
- Labels are used to select the kubernetes internal resources 

**Annotations** 
- It has no certain limits and we can use special characters
- Annotations are  used to select the external resources like ingress controller 
- We can keep some metadata like buildURL

![alt text](img/ingress.drawio.svg)


