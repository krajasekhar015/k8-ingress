# Liveness Probe & Readiness Probe

One of the feature of kuberenetes is self healing. It means auto restart 
- If your application is not working, kubernetes can auto restart your application

**How would kuberenetes know that the application is working or not?**
A) This is through liveness probe and readiness probe

- liveness probe --> health check 
- readiness probe --> ready to accept traffic 

- When port 8080is opened, then we can say mysql is ready 
- When the application gets ready, two things are important. They are; 
    - mentioning the resources 
    - configuring the liveness probe and readiness probe 

In expense-k8 -> mysql -> manifest.yaml 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: expense
  labels:
    app: mysql
    tier: db
    project: expense
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mysql
      tier: db
      project: expense
  template:
    metadata:
      labels:
        app: mysql
        tier: db
        project: expense
    spec:
      containers:
      - name: mysql
        image: mahalakshmi2997/mysql:v1
        readinessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 15
          periodSeconds: 10
---
kind: Service
apiVersion: v1
metadata:
  name: mysql
  namespace: expense
spec:
  selector:
    app: mysql
    tier: db
    project: expense
  ports:
  - name: mysql-port
    protocol: TCP
    port: 3306 # service port
    targetPort: 3306 # container port
```

```
cd expense-k8/ 
```
```
kubectl apply -f namespace.yaml 
```
```
kubens expense 
```
```
cd mysql/ 
```
```
kubectl apply -f manifest.yaml 
```
```
kubectl get pods 
```

In backend -> manifest.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend
  namespace: expense
data:
  DB_HOST: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: expense
  labels:
    app: backend
    tier: api
    project: expense
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
      project: expense
  template:
    metadata:
      labels:
        app: backend
        tier: api
        project: expense
    spec:
      containers:
      - name: backend
        image: mahalakshmi2997/backend:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          # limits is greater than or equal to requests
          limits:
            cpu: 100m
            memory: 128Mi
        envFrom:
        - configMapRef:
            name: backend
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3
---
kind: Service
apiVersion: v1
metadata:
  name: backend
  namespace: expense
spec:
  selector:
    app: backend
    tier: api
    project: expense
  ports:
  - name: backend-port
    protocol: TCP
    port: 8080 # service port
    targetPort: 8080 # container port
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
 name: backend
 namespace: expense
spec:
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: backend
 minReplicas: 1
 maxReplicas: 10
 targetCPUUtilizationPercentage: 15 # usually 75 in real environment
```

```
cd ../backend/
```
```
kubectl apply -f manifest.yaml 
```
```
kubectl get pods 
```

# Init Containers

- Before backend starts, we need to make sure DB is running and accessable. 
- Before running backend, init containers will check whether MySql is running or not
- we can run init containers before main containers run. It can be one or many 
- init containers should be ready before main containers runs 
- If init containers fails, main container will not run 
- There is no completion state for our containers. Init containers goes to completion state 
- To set configuration and check external dependency applications status, we can make use of init containers

example:
----------
for i in {1..100}; do sleep 1; if nslookup mysql; then exit 0; fi;done;exit 1 

In expense-k8 --> backend -> manifest.yaml 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend
  namespace: expense
data:
  DB_HOST: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: expense
  labels:
    app: backend
    tier: api
    project: expense
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
      project: expense
  template:
    metadata:
      labels:
        app: backend
        tier: api
        project: expense
    spec:
      initContainers:
      - name: mysql-check
        image: busybox:1.28
        command: ['sh', '-c', "until nslookup mysql; do echo waiting for myservice; sleep 2; done"]
      containers:
      - name: backend
        image: mahalakshmi2997/backend:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          # limits is greater than or equal to requests
          limits:
            cpu: 100m
            memory: 128Mi
        envFrom:
        - configMapRef:
            name: backend
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3
---
kind: Service
apiVersion: v1
metadata:
  name: backend
  namespace: expense
spec:
  selector:
    app: backend
    tier: api
    project: expense
  ports:
  - name: backend-port
    protocol: TCP
    port: 8080 # service port
    targetPort: 8080 # container port
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
 name: backend
 namespace: expense
spec:
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: backend
 minReplicas: 1
 maxReplicas: 10
 targetCPUUtilizationPercentage: 15 # usually 75 in real environment
```

```
kubectl apply -f manifest.yaml 
```
```
kubectl get pods
```


**How to use configMap as volume?**
- Application code and Configuration are independent 
- So, we need to set for less number of builds
- Due to changes in configuration, we should not get rebuild of applications


In expense-k8 --> frontend -> manifest.yaml 

We will copy the frontend config file from expense docker and pasted in the starting of ConfigMap

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend
  namespace: expense
data:
  nginx.conf: | # | represents multi line file
    user www-data;
    worker_processes 4;
    pid /var/run/nginx.pid;

    events {
      worker_connections 768;
      # multi_accept on;
    }

    http {

      ##
      # Basic Settings
      ##

      sendfile on;
      tcp_nopush on;
      tcp_nodelay on;
      keepalive_timeout 65;
      types_hash_max_size 2048;
      large_client_header_buffers 6 32k;
      client_max_body_size 100m;

      # server_names_hash_bucket_size 64;
      # server_name_in_redirect off;
      include /etc/nginx/mime.types;
      default_type application/octet-stream;

      ##
      # Logging Settings
      ##
      access_log /var/log/nginx/access.log;
      error_log /var/log/nginx/error.log debug; # change from debug to warn or error for production

      ##
      # Gzip Settings
      ##
      gzip on;
      gzip_disable "msie6";

      ##
      # Virtual Host Configs
      ##

      include /etc/nginx/conf.d/*.conf;
      include /etc/nginx/sites-enabled/*;

      server {
        listen       80;
        server_name  localhost;

        proxy_http_version 1.1;

        #charset koi8-r;
        #access_log  /var/log/nginx/host.access.log  main;
        #error_log /dev/stdout debug;
        #rewrite_log on;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            ssi    on;
        }

        location /images/ {
            expires 5s;
            root   /usr/share/nginx/html;
            try_files $uri /images/placeholder.png;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }

        location /api/ { 
            proxy_pass http://backend:8080/;
        }

        }

    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: expense
  labels:
    app: frontend
    tier: web
    project: expense
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: web
      project: expense
  template:
    metadata:
      labels:
        app: frontend
        tier: web
        project: expense
    spec:
      containers:
      - name: frontend
        image: mahalakshmi2997/frontend:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          # limits is greater than or equal to requests
          limits:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
              readOnly: true
      volumes:
      - name: nginx-conf
        configMap:
          name: frontend
          items:
            - key: nginx.conf
              path: nginx.conf
---
kind: Service
apiVersion: v1
metadata:
  name: frontend
  namespace: expense
spec:
  type: LoadBalancer
  selector:
    app: frontend
    tier: web
    project: expense
  ports:
  - name: frontend-port
    protocol: TCP
    port: 80 # service port
    targetPort: 80 # container port
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
 name: frontend
 namespace: expense
spec:
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: frontend
 minReplicas: 1
 maxReplicas: 10
 targetCPUUtilizationPercentage: 15 # usually 75 in real environment
```

```
cd ../frontend/ 
```
```
kubectl apply -f manifest.yaml 
```
```
kubectl get configmap 
```
```
kubectl get deployments 
```
```
kubectl get pods 
```
```
kubectl describe pod <pod-name>
```
```
kubectl exec -it <pod-name> -- bash 
```
```
cd /etc/nginx/
```
```
cat nginx.conf
```

- Now, we will make some changes in config-map. Lets us make a comment in the config file as `# this is as part of nginx configmap` at the last of the config file

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend
  namespace: expense
data:
  nginx.conf: | # | represents multi line file
    user www-data;
    worker_processes 4;
    pid /var/run/nginx.pid;

    events {
      worker_connections 768;
      # multi_accept on;
    }

    http {

      ##
      # Basic Settings
      ##

      sendfile on;
      tcp_nopush on;
      tcp_nodelay on;
      keepalive_timeout 65;
      types_hash_max_size 2048;
      large_client_header_buffers 6 32k;
      client_max_body_size 100m;

      # server_names_hash_bucket_size 64;
      # server_name_in_redirect off;
      include /etc/nginx/mime.types;
      default_type application/octet-stream;

      ##
      # Logging Settings
      ##
      access_log /var/log/nginx/access.log;
      error_log /var/log/nginx/error.log debug; # change from debug to warn or error for production

      ##
      # Gzip Settings
      ##
      gzip on;
      gzip_disable "msie6";

      ##
      # Virtual Host Configs
      ##

      include /etc/nginx/conf.d/*.conf;
      include /etc/nginx/sites-enabled/*;

      server {
        listen       80;
        server_name  localhost;

        proxy_http_version 1.1;

        #charset koi8-r;
        #access_log  /var/log/nginx/host.access.log  main;
        #error_log /dev/stdout debug;
        #rewrite_log on;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            ssi    on;
        }

        location /images/ {
            expires 5s;
            root   /usr/share/nginx/html;
            try_files $uri /images/placeholder.png;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
        # this is as part of nginx configmap
        location /api/ { 
            proxy_pass http://backend:8080/;
        }

        }

    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: expense
  labels:
    app: frontend
    tier: web
    project: expense
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: web
      project: expense
  template:
    metadata:
      labels:
        app: frontend
        tier: web
        project: expense
    spec:
      containers:
      - name: frontend
        image: mahalakshmi2997/frontend:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          # limits is greater than or equal to requests
          limits:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
              readOnly: true
      volumes:
      - name: nginx-conf
        configMap:
          name: frontend
          items:
            - key: nginx.conf
              path: nginx.conf
---
kind: Service
apiVersion: v1
metadata:
  name: frontend
  namespace: expense
spec:
  type: LoadBalancer
  selector:
    app: frontend
    tier: web
    project: expense
  ports:
  - name: frontend-port
    protocol: TCP
    port: 80 # service port
    targetPort: 80 # container port
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
 name: frontend
 namespace: expense
spec:
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: frontend
 minReplicas: 1
 maxReplicas: 10
 targetCPUUtilizationPercentage: 15 # usually 75 in real environment
```

```
kubectl apply -f manifest.yaml
```
```
kubectl get pods 
```
- Now, lets us check if we delete both the frontend pods, then it will be created automatically and the config map remains in it

```
kubectl delete pod <pod-1> <pod-2>
```
```
kubectl get pods 
```
```
kubectl exec -it <pod-name> -- bash 
```
```
cd /etc/nginx/
```
```
cat nginx.conf
```