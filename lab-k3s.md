# Lab - K3s

## Preparation

To interact with the K3s cluster using a GUI install <u>OpenLens v6.2.5</u>. The `config` file to access the cluster is provided with the lab files.

It is possible to interact with the cluster using the CLI  (remember to use the VM).

```shell
kubectl get nodes
kubectl get pods --all-namespaces
kubectl create namespace nmec
kubectl delete namespace nmec
```

## 1. Introduction

### Build a Docker image and apply the deployment

```shell
docker build -t registry.deti/nmec/app:v1 .
docker push registry.deti/nmec/app:v1
kubectl apply -f deployment.yaml [-n nmec] # -n is namespace (optional)
```

The error `ImagePullBackOff` means that the Docker image is not in the registry.

To access the website, go to OpenLens - Forward - Open in Browser.

## 2. Service

A service can be used to foward traffic from a specific port in the cluster to a port in the pod.

```yml
apiVersion: v1
kind: Service
metadata:
  name: app
  namespace: nmec
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: app-v2 # label of deployment
```

In this example, a service is used to forward traffic from <u>port 80</u> in the cluster to <u>port 8080</u> in the pod. Using labels allows us to connect pods to services. The number of replicas corresponds to the number of pods in the service.

## 3. Routing

Ingress allows us to connect to services from the Internet.

```yml
apiVersion: v1
kind: Ingress
metadata:
  name: app-nmec-k3s
  namespace: nmec
  annotations:
    spec.ingressClassName: traefik
    traefik.ingress.kubernetes.io/frontend-entry-points: http,https
    traefik.ingress.kubernetes.io/redirect-entry-point: https
    traefik.ingress.kubernetes.io/redirect-permanent: "true"
spec:
  rules:
  - host: app-nmec.k3s
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app
            port: 
              number: 80
```

In this example, the app should be accessible on a browser at `http://app-nmec.k3s`.

Note: if `app-nmec.k3s` cannot be accessed, add the entry `193.136.82.36 app-nmec.k3s` to `/etc/hosts` (Ubuntu) or to `C:/Windows/System32/drivers/etc/hosts` (Windows).

## 4. Load Balancer

The normal load balancer distributes traffic among the deployment pod replicas.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: nmec
spec:
  replicas: 10 # app has 10 replicas
```

## 5. Custom LB

Using nginx to serve as a load balancer for our app, we are able to customize the load balancing strategy and our app is not directly exposed to the Internet.

```nginx
http {
  ...
  server {
    listen 80 default_server;
    server_name app.k3s;
    location / {
      proxy_pass http://app:8080/;
    }
  }
}
```

We only need to create a service and a deployment for nginx. We should include in the nginx Docker image the routing configurations, where nginx points traffic to `http://app:8080` - always use the <u>K3s label</u> and not an IP address!

## 6. Volumes

Volumes are used to store static files that can be shared among multiple containers.

```shell
kubectl apply -f storage.yaml [-n nmec]
kubectl cp -n nmec ./www/img.gif nginx-POD-ID:/var/www/static # Copy file
```

Nginx can be used to redirect static requests to a location in the volume.

```nginx
http {
  ...
  server {
    listen 80 default_server;
    server_name app.k3s;
    location / {
      proxy_pass http://app:8080/;
    }    
    location /static {
      root /var/www;
    }
  }
}
```

## 7. Configuration

ConfigMaps can be injected in a deployment. It exposes a name that points to a volume. This way, we separate configuration from the Docker image - no need to re-build the image when the configuration changes.

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: nmec
data:
  nginx.conf: |
    user  nginx;
    worker_processes  1;
    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;
    events {
      worker_connections  1024;
    }
    http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;
      sendfile      on;
      server {
        listen 80 default_server;
        server_name app.k3s;
        location / {
          proxy_pass http://app:8080/;
        }
        location /static {
          root /var/www;
        }
      }
    }
```

In this example, we inject a new configuration for the nginx instances.

## 8. Secrets

Secrets are like ConfigMaps, but are used to store sensitive information like SSL certificates for HTTPS, access credentials or API keys.

```shell
kubectl create secret generic app-secret --from-file=secret -n nmec
```

After generating the secret in the cluster, we can reference it in the deployment.

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: nmec
spec:
  replicas: 5
  ...
  template:
    ...
    spec:
      containers:
      ...
      volumes:
        - name: app-secret
          secret:
            secretName: app-secret
```

## 9. Affinity

Affinity rules can be used to prevent replicas of a same deployment to run on the same machine/node. They allow us to specify a node to run a replica or that a pair of replicas must run in the same node (like an app and the database). 

Anti-affinity means replicas never run together.

```yml
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nmec
spec:
  replicas: 5
  ...
  template:
    ...
    spec:
      containers:
      ...
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: nginx
```

In this example, we impose that nginx replicas cannot be running the same node.

## 10. Autoscaler

Allows scalling of replicas, based on resource usage. We can specify a minimum and a maximum number of replicas for each deployment.

```shell
kubectl create autoscale deployment app --cpu-percent=10 --min=1 --max=20
```
