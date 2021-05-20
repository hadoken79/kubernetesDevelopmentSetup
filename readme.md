# kubernetes Demo
cluster is setup in minikube

To get this demo running
the deplayments have to be created in the correct order.
feg. in mongo-express.yaml certain variables are set from a configmap.
So this configma must extist first, before the deployment for mongo-express can be created

to list running deployments 
    kubectl get deployment

to list services 
    kubectl get services

...pods 
    kubectl get pods

by default only ressources in 'default' namespace are listet.  
to get elements from specific namespace (example for ingress controller wich is in 'kube-system' namespace) 
    kubectl get pod -n kube-system

To create deployments use kubectl
    kubectl apply -f mongo-configmap.yaml
apply can be used for new creations and for updates.

In this yaml files, definitions for Deployments and corespondent services are definded in same file.

To make a kluster external aviable either e external service has to be defined (ok for development)
this can be made, by adding the 'loadbalancer' attribute to the service specs and to provide an external port
```yaml
spec:
  selector:
    app: mongo-express
  type: LoadBalancer #for external availability of this service: this type accepts external requests by assigning externap ip to service
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
    nodePort: 30000 #for external availability of this service: external port for entering kluster. has to be set by definition between 30000 - 32767
```

In a production Enviroment it is better to do the routing with ingress (needs to be installed... there are many different options also one from k8s with nginx)
to activate ingress in minikube: (which uses the official k8s ngnix ingress controller) 
    minikube addons enable ingress

Ingress controller routes external request then to ingress pod, which connects to defined "internal" services

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
 # namespace: kube-system #not needed, because for this example the mongo-express service and pod are in "default" namespace
spec:
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
now for testing purpouses map the Ingress IP in /etc/hosts to dashboard.com
to get the Ingress-IP:
    kubectl get Ingress [--watch]
if Ingress is in certain namespace
    kubectl get Ingress -n [mynamespace]` | `kubectl get Ingress -n [mynamespace]`
`dashboard-ingress   <none>   dashboard.com   192.168.49.2   80      6m44s`

