# FleetMan web application

Application developed in Java spring boot on the backend and Angular js on the frontend

## Architecture used:

![architecture diagram](assests/architecture.drawio.png)

deployment done in kubernetes

## Managing production level kubernetes

### Kops
**pros**
* Kops is well respected and heavily used
* it's easy to use

**cons**
* You are responsible for managing the master node
* By default you get only a single master node
* it feels like more work than EKS

### EKS
**pros**
* it has gained a lot of ground recently
* using eksctl, it is quite simple to use
* No management of the master required

**cons**
* it needs a third party tool to make it usable
* The GUI is (time of writing this) very poor, usage of cli is better
* might feel like you're tied to AWS


## Advanced topics in k8s

#### Resource Request and limits

Used in defining the amount of CPU and memory resources that a container requires. These settings play a crucial role in the scheduling and resource allocation of pods.

example
 ```
  resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
 ```


#### Horizontal pod autoscaling

Horizontal Pod Scaling allows you to automatically scale the number of pods in a deployment based on observed CPU utilization or other custom metrics. To enable horizontal pod scaling, you can modify the deployment definition.

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

This example sets up a Horizontal Pod Autoscaler for the webapp deployment, targeting 50% CPU utilization. It will scale the number of replicas between 2 and 5 based on observed CPU usage.

#### Liveness and readiness probes

Liveness and readiness probes help Kubernetes determine the health and availability of a pod.
Let's add liveness and readiness probes to the webapp deployment:

```
  livenessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /readiness
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5        
```

In this example, a liveness probe checks the /health endpoint every 10 seconds, starting 30 seconds after the container starts. If the probe fails, Kubernetes will restart the container.

The readiness probe checks the /readiness endpoint every 5 seconds, starting 5 seconds after the container starts. If the readiness probe fails, the pod is temporarily marked as not ready, preventing it from receiving traffic until it passes again.

#### configMap and secrets

In Kubernetes, ConfigMaps and Secrets are used to manage configuration settings and sensitive information, respectively, separate from the application code.

##### configMap

It allows you to decouple configuration details from the container images. For example, you can store environment variables, configuration files, or any non-sensitive data in a ConfigMap. To use ConfigMap in your deployment, reference it in your pod's specification.

Example of using a ConfigMap in a deployment:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  config.properties: |
    key1=value1
    key2=value2
```

Reference in a pod

```
spec:
  containers:
  - name: my-container
    envFrom:
    - configMapRef:
        name: my-config
```

##### Secrets

Similar to ConfigMaps, Secrets store sensitive information such as passwords or API tokens. They are base64-encoded, but still, it's important to handle them carefully. Reference secrets in your pod's specification.

Example of using a Secret in a deployment:

```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
data:
  username: dXNlcm5hbWU=
  password: cGFzc3dvcmQ=

```

Reference in a pod:

```
spec:
  containers:
  - name: my-container
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: username

```

#### Quality of Service and Eviction

In Kubernetes, Quality of Service (QoS) classes are assigned to pods based on their resource requirements. This information helps Kubernetes make eviction decisions when the system is under resource pressure.

QoS Classes: There are three QoS classes: Guaranteed, Burstable, and BestEffort.

Guaranteed: Pods with specified resource requests and limits for both CPU and memory. They are assured to get the resources they request.

Burstable: Pods with resource requests, but without specifying resource limits. They can use resources beyond their requests if available.

BestEffort: Pods without specifying resource requests or limits. They get whatever resources are available.

Eviction: When a node is under resource pressure, Kubernetes might evict pods to ensure the overall stability of the cluster. Eviction decisions are influenced by QoS classes and priorities. Pods with lower QoS classes are more likely to be evicted first.

Example of setting QoS in a pod:

```
spec:
  containers:
  - name: my-container
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"

```


#### Ingress Controller Overview

In Kubernetes, an Ingress Controller acts as an intelligent entry point for external traffic to your cluster, providing HTTP and HTTPS routing to services based on defined rules. It simplifies and centralizes the management of external access to services, making it easier to expose multiple applications and services through a single entry point.

##### The Problem It Solves

Without an Ingress Controller, managing external access to services can be cumbersome. Each service would require its own LoadBalancer or NodePort, making it difficult to scale and manage as the number of services grows. Ingress Controllers solve this problem by providing a unified and flexible way to handle external traffic.

###### Setup
Ingress Resource

The Ingress resource in Kubernetes defines how external HTTP/S traffic should be routed to your services. It acts as a configuration file for the Ingress Controller, specifying rules for routing requests based on hostnames, paths, and other criteria.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: mydomain.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: my-api-service
                port:
                  number: 8080
  tls:
    - hosts:
        - mydomain.com
      secretName: my-tls-secret

```

* metadata: Specifies metadata for the Ingress resource, including its name.
* spec: Defines the rules for routing traffic.
  * rules: Specifies the hostnames and their associated paths.
    * Requests to mydomain.com/app are directed to my-app-service.
    * Requests to mydomain.com/api are directed to my-api-service.
  * tls: Optional block for enabling TLS/SSL termination.
    * hosts: Specifies the host for which TLS is enabled.
    * secretName: References the Kubernetes Secret containing the TLS certificate and key.


Ingress Controller

The Ingress Controller is a pod running in your Kubernetes cluster responsible for implementing the rules defined in the Ingress resource. It interprets the Ingress rules and configures the underlying load balancer to direct traffic accordingly.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ingress-nginx
  template:
    metadata:
      labels:
        app: ingress-nginx
    spec:
      containers:
        - name: ingress-nginx-controller
          image: k8s.gcr.io/ingress-nginx/controller:v1.2.1
          args:
            - /nginx-ingress-controller
            - --publish-service=ingress-nginx-controller
            - --election-id=ingress-controller-leader
            - --ingress-class=nginx

```

* metadata: Specifies metadata for the Deployment, including its name and namespace.
* spec: Defines the deployment specification.
  * replicas: Number of Ingress Controller pods to run.
  * selector: Matches the labels of the Ingress Controller pods.
  * template: Defines the pod template.
    * containers: Specifies the Ingress Controller container.
    * image: Docker image for the Nginx Ingress Controller (k8s.gcr.io/ingress-nginx/controller:v1.2.1).
    * args: Command-line arguments for the Ingress Controller, specifying configuration options like the ingress class, election ID, and publish service.

###### Summary
Ingress Resource:

* Acts as a configuration file for the Ingress Controller.
* Specifies rules for routing external HTTP/S traffic.
* Defines TLS/SSL configurations if needed.

Ingress Controller:

* Implements Ingress rules by configuring the underlying load balancer.
* Runs as a pod in the Kubernetes cluster.
* Interprets and applies rules defined in the Ingress resource.

##### Main Functions of Ingress Controllers

###### Routing External Traffic:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: mydomain.com
      http:
        paths:
          - path: /frontend
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /backend
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80

```

Routes external HTTP traffic based on rules defined in the Ingress resource.

Requests to mydomain.com/frontend are directed to the frontend service, and requests to mydomain.com/backend are directed to the backend service.

###### TLS/SSL Termination

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: mydomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
  tls:
    - hosts:
        - mydomain.com
      secretName: my-tls-secret
```

Enables termination of TLS/SSL encryption at the Ingress Controller.

The tls block specifies the host and the secret containing the SSL certificate.

###### Path-Based Routing

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: mydomain.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: my-api-service
                port:
                  number: 8080
```

Routes traffic based on the path of the URL.

Requests to mydomain.com/app are directed to the my-app-service, and requests to mydomain.com/api are directed to the my-api-service

###### TCP and UDP Load Balancing

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-tcp-ingress
  namespace: ingress-nginx
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "TCP"
spec:
  rules:
  - host: mydomain.com
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend
            port:
              number: 80
  tcp:
  - port: 5000
    backend:
      service:
        name: tcp-service
        port:
          number: 5000

```

Facilitates TCP/UDP load balancing for non-HTTP services.

The tcp block specifies a backend service (tcp-service) and port (5000) for TCP traffic.

###### Rewriting and Redirection

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: mydomain.com
    http:
      paths:
      - path: /old-path
        pathType: Prefix
        backend:
          service:
            name: old-service
            port:
              number: 80
        pathType: Prefix
        pathRewrite:
          rewriteTarget: /new-path
```

Supports URL path rewriting and redirection.

Requests to mydomain.com/old-path are rewritten to mydomain.com/new-path before being directed to the old-service.

* Role based access controls
* jobs, Daemonsets and stateful sets
* CI/CD
