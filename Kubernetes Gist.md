# Kubernetes Gist

## basic kubectl commands

every recource lives in its namespace.  
to list them, the optional namespace parameter has to be provied in command  
if no namespace parameter is provided, kubectl checks in "default" namespace  

to list pods

    kubectl get pod[s] [-n mynamespacename]

to get more nformation  

    kubectl get pod <podname> -o wide

for all pods

    kubectl get pod -o wide

pods are connected with a internal or external service  
to list services  

    kubectl get services [-n optionalnamespace  

to get more information

    kubectl describe ingress [ingressname]

to create a deployment, service or ingress: (apply can be used for creation and update "-f" is for referencing a file)

    kubectl apply -f myfile.yaml

to remove/destroy a pod/deployment

    kubectl delete -f myfile.yaml

## Pod/deployment definition

a pod lives in a deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rederecter
  namespace: default
  labels:
    app: rederecter
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: rederecter
  template:
    metadata:
      labels:
        app: rederecter
    spec:
      containers:
      - name: rederecter
        image: morbz/docker-web-redirect
        ports:
        - containerPort: 80
        env:
        - name: REDIRECT_TARGET
          value: https://bing.com
```

## handle enviromental variables and secrets
if a container in a pod needs access to env variables or f.e. database secrets  
its best to use k8s configmap for variables and secrets for credentials

example of a configmap deployment
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
  namespace: default # here can namespace be declared in which the ressource is created and accessible. for default this doesn't need to be defined here. this is just an example. (volumes can not be created in namespaces. they are global accessable)
data:
  database_url: mongodb-service # to access a service from another namespase, namespace name can be attached after a dot. mongodb.service.myNamespace
```

example of a secret  
to encode a value base64 in linux-termal

    echo -n 'test' | base64

```yaml
apiVersion: v1 #secrets should not be defined in configmaps. Here in Secret data can be encoded (here base64)
kind: Secret
metadata:
  name: mongodb-secret
type: opaque
data:
  mongo-root-username: dGVzdHVzZXI= # base64
  mongo-root-password: dGVzdHBhc3N3b3Jk # base64
```

these can then be accessed in pods, and are available inside as enviromental variables. example of secret  
```yaml
#Definition for Pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username #testuser
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password #testpassword
```
...example for configmap
```
...
spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom: #from configmap element in k8s
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url
```

## Services
pods communicate and are accessed through services. most of the time   it is best to defines those in the same file as the pod  
with three "---" kubernetes sees theese sections as separate files

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rederecter
  namespace: default
  labels:
    app: rederecter
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: rederecter
  template:
    metadata:
      labels:
        app: rederecter
    spec:
      containers:
      - name: rederecter
        image: morbz/docker-web-redirect
        ports:
        - containerPort: 80
        env:
        - name: REDIRECT_TARGET
          value: https://bing.com
---
# Default redirect Service
apiVersion: v1
kind: Service
metadata:
  name: rederecter-service
  namespace: default
spec:
  selector:
    app: rederecter
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```
by default services are internal, you can make a service external availabe by adding the "loadbalancer" attribute to it  
however for production it is better to use ingress. (kubectl describe service, will show the if address)
```yaml
spec:
  selector:
    app: mongo-express
  type: LoadBalancer #for external availability of this service: this type accepts external requests by assigning external ip to service
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
    nodePort: 30000 #for external availability of this service: external port for entering kluster. has to be set by definition between 30000 - 32767
```
When the services has to use multiple target ports, the port definitions need to be named.  
if just a single port is used, name parameter is optional.  
```yaml
...
  ports:
  - name: mongodb
    protocol: TCP
    port: 27017
    targetPort: 27017
  - name: mongodb_exporter
    protocol: TCP
    port: 9216
    targetPort: 9216
...
```

## ingress
For routing, loadbalancing and more Ingress can be used to connect app to the internet  
there are many options for defferent ingress variants. this is en example of k8s-nginx (default backend is in this example not worng correctly)
it listens for host dashboard.com. (must of course be defined in /hosts of dns)  
host can be let aside, the all traffic to this ingress will be checked for defined paths  
in that case, delete the host attribute and add a "-" to http (this is now the list element)

> watch yout for apiVersion. the definitions for syntax in file are often changing with version

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress-resource
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/add-base-url: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  defaultBackend:
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
  - host: dashboard.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mongo-express-service
            port: 
              number: 8081
      - path: /red
        pathType: Prefix
        backend:
          service:
            name: rederecter-service
            port: 
              number: 80
      - path: /test
        pathType: Prefix
        backend:
          service:
            name: mongo-express-service
            port: 
              number: 8081
```

## TLS Certificates
these can easely be managed in kubernetes
store the credentials in a secret and refrence them in ingress:

https://www.youtube.com/watch?v=X48VuDVv0do&t=8410s
timestamp: 2:23:17

just define a secret wich holds the cert and reference in "tls section" under spec: within Ingress

Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/add-base-url: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts: 
    - dashboard.com
    secretName: myapp-secret-tls
  rules:
  - host: dashboard.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mongo-express-service
            port: 
              number: 8081
```
in kubernetes there is a specyfic type for tls secrets. use this type attribute  
this ts secret has to be in same namespace as the ingress wich references it.
tlsSecret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret-tls
  namespace: default
data:
  tls.crt: base64encoded cert
  tls.key: base64encoded key
type: kubernetes.io/tls
```

## Helm
k8s package-manager  

Helm-Charts (cluster of yaml files)  
These are avaibalbe via search

    helm search <packagename>
  
or on [Helm hub:](https://hub.helm.sh/)  

Helm can also be used as template engine  
f.e. if many microservices are deployed, and there are only minor differences in servicefiles  
then helm can be handy, so there is no need to create many yaml files, but just a blueprint.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{.Values.name}}
spec:
  containers:
    - name: {{.Values.container.name}}
      image: {{.Values.container.image}}
      port: {{.Values.container.port}}
```
values.yaml
```yaml
name: my-app
container:
  name: my-app-container
  image: my-image
  port: 9001
```

The structure of a Helm-Chart

> mychart/
>   Chart.yaml
>   values.yaml
>   charts/
>   templates/

Toplevel mychart -> name of the chart  
Chart.yaml -> metadate of the chart  
values.yaml -> values for the template files  
chartsfolder -> dependencies  
templates -> the actual temlate file  

When running hel install, the values from values.yaml file are injected in template  
Default values from default file can be overridden during installation

    helm install --values=my-own-values.yaml <chartname>

these values will then be merged. so you don't have to prevent all values, only those who are new or should be overridden.  
For simple attributes, values can also be passed direct

    helm install --set version=2.0.0

## Volumes

There are 3 elements that are used to persist data in kubernetes

- persistent volume
- persistent volume claim
- storage class

Volumes are not namespaced, they are accessible in the whole cluster  
For database persistence use always remote storage. Local volumes are tied to pods and aren't crash safe  

the interface to a volume can be defined in a yaml file. The recourse itself has to be present already, when the cluster is created.  
this has to be done via the administrator... (user can deploy aplications but admin has to provide recourses..)  
with external defined storage. the developer has to claim the existing storage (p.v) f.e. cloud volume with storage-claim (yaml)  

this claim has then to be referenced in the pod who uses the storage. (claims must exist in same namespace as pod)

So to sum up:
> a volume has to be created (cloud, nfs, local etc)
> the admin has to create a persistence volume definition
> the user has to claim a volume, with a volume-claim
> this claim connects to a volume that matches the definition
> inside a pod now a volume can be defined and mountet

example:
pod  
```yaml
apiVersion: v1
kind: pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
      name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: pvc-name
 ```
 pvc  
 ```yaml
kind: persistentVolumeClaim
apiVersion: v1
metadata:
  - name: pvc-name
spec:
  storageClassName: manual
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  recourses:
    requests:
      storage: 10Gi
 ```

 Configmaps and Secrets are volumes too, an can be mountet the same way, instead of env variables

 ```yaml
spec:
  containers:
  - image: elastic:latest
    name: elastic-container
    ports:
    - containerPort: 9200
    volumeMounts:
    - name: es-persistent-storage
      mountPath: /var/lib/data
    - name: es-secret-dir
      mountPath: /var/lib/secret
    - name: es-config-dir
      mountPath: /var/lib/config
  volumes_
  - name: es-persistent-storage
    persistentVolumeClaim:
      claimName:es-pv-claim
  - name: es-secret-dir
    secret:
      secretName: es-secret
  - name: es-config-dir
    configMap:
      name: es-config-map
 ```

 ## storage class
 above workflow can get messy, when many pods use many storage object and those change often.  
 with storage class volumes can be created dynamically, when a pvc claims one

 Storage can be defined also in a yaml file
my-storageClass.yaml
```yaml
apiVersion: storage.k8s.io/v1
kind: storageClass
metadata:
 name: storage-class-name
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```
the provisioner is unique for different storage providers like aws, google-cloud and aws f.e.  
this volume can now also be claimed via pvc.  
therefore an aditional attribute in pvc has to be made "storageClassName"
```yaml
apiVersion: v1
kind: persistentVolumeClaim
metadata: 
  name: mypvc
spec:
  accessModes:
  - ReadWriteOnce
  recourses:
    requests:
      storage: 100Gi
  storageClassName: storage-class-name
```

## Statefull Set

For statefull applications "delpoyment" isn't the right choise.  
In that case use "statfull sets" this allows for managin a cluster f.e. mysql pods.  
One will be the master who can write to the storage and all the others will be workers, who only can read.  
In a statefull set the identity of a pod is fixed (name and dns) so when one dies it can take the same place to the same state,   
as the previous was. evey worker will automaticaly synchronize the data from the previous pod in line.

## Different Services in K8s

### Cluster IP Service

This is the default Service, if no type is specified
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
...
```
### Headless Service

In cases where pods need to cummuinicate directly and not throu a service  
A Headless service can be used. Normaly when a client makes a dns-lookup call,   
the clusterIP of the service is returned. A Headless service has no clusterIP asigned  
and in this case the clusterIPs of the pods are returned. no they can address each other directly. 
A typical usecase whould be a database, where a regular "clusterIP service" is used to access the database-service,  
f.e. for node to connect to db, and a "Headless Service" aditional.  
So the worker pods can directly connect to previous, or masternodes, without going to service first.  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  clusterIP: None
  selector:
  ...
```









