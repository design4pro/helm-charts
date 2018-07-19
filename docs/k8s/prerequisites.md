## Create service account for helm

First create a service account and attach cluster-admin role to it. This enables the tiler pod to communicate with the kubernetes api. There are reasons why you should do this[^1]. This can be done with `kubectl apply -f <file>`

``` docker
# This is an extract from here: http://jayunit100.blogspot.fi/2017/07/helm-on.html
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: helm
    namespace: kube-system
```

## Deploy NGINX Ingress Controller with RBAC enabled

If your Kubernetes cluster has RBAC enabled, from the Cloud Shell, deploy an NGINX controller Deployment and Service by running the following command:

``` bash
helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true --namespace kube-system
```

## Deploy NGINX Ingress Controller with RBAC disabled

If your Kubernetes cluster has RBAC disabled, from the Cloud Shell, deploy an NGINX controller Deployment and Service by running the following command:

``` bash
helm install --name nginx-ingress stable/nginx-ingress
```

In the ouput under `RESOURCES`, you should see the following:

``` bash
==> v1/Service
NAME                           TYPE          CLUSTER-IP    EXTERNAL-IP  PORT(S)                     AGE
nginx-ingress-controller       LoadBalancer  10.7.248.226  pending      80:30890/TCP,443:30258/TCP  1s
nginx-ingress-default-backend  ClusterIP     10.7.245.75   none         80/TCP                      1s
```

Wait a few moments while the GCP L4 Load Balancer gets deployed. Confirm that the nginx-ingress-controller Service has been deployed and that you have an external IP address associated with the service. Run the following command:

``` bash
kubectl get service nginx-ingress-controller
```

``` bash
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-controller   LoadBalancer   10.7.248.226   35.226.162.176   80:30890/TCP,443:30258/TCP   3m
```

Notice the second service, nginx-ingress-default-backend. The default backend is a Service which handles all URL paths and hosts the NGINX controller doesn't understand (i.e., all the requests that are not mapped with an Ingress Resource). The default backend exposes two URLs:

*/healthz that returns 200
*/ that returns 404

## Configure Ingress Resource to use NGINX Ingress Controller

An Ingress Resource object is a collection of L7 rules for routing inbound traffic to Kubernetes Services. Multiple rules can be defined in one Ingress Resource or they can be split up into multiple Ingress Resource manifests. The Ingress Resource also determines which controller to utilize to serve traffic. This can be set with an annotation, `kubernetes.io/ingress.class`, in the metadata section of the Ingress Resource. For the NGINX controller, use the value nginx as shown below:

``` docker
annotations: kubernetes.io/ingress.class: nginx
```

On Kubernetes Engine, if no annotation is defined under the metadata section, the Ingress Resource uses the GCP GCLB L7 load balancer to serve traffic. This method can also be forced by setting the annotation's value to `gce` as shown below:

``` docker
annotations: kubernetes.io/ingress.class: gce
```

Lets create a simple Ingress Resource YAML file which uses the NGINX Ingress Controller and has one path rule defined by typing the following commands:

``` bash
touch ingress-resource.yaml
nano ingress-resource.yaml
```

Copy the contents of ingress-resource.yaml[^2] into the editor, then press `Ctrl-X`, then press `y`, then press `Enter` to save the file.

The `kind: Ingress` dictates it is an Ingress Resource object. This Ingress Resource defines an inbound L7 rule for path `/hello` to service `hello-app` on port 8080.

From the Cloud Shell, run the following command:

``` bash
kubectl apply -f ingress-resource.yaml
```

Verify that Ingress Resource has been created. Please note that the IP address for the Ingress Resource will not be defined right away (wait a few moments for the ADDRESS field to get populated):

``` bash
kubectl get ingress ingress-resource
```

``` bash
NAME               HOSTS     ADDRESS   PORTS     AGE
ingress-resource   *                   80        `
```

[^1]: Why helm needs a service account - http://jayunit100.blogspot.fi/2017/07/helm-on.html
[^2]

``` docker
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-resource
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /hello
        backend:
          serviceName: hello-app
          servicePort: 8080
```
