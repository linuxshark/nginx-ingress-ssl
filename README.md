# **Nginx Ingress controller with LetsEncrypt automatic SSL cert**

## steps

* 1.- Install helm and tillers

* 2.- Install nginx-ingress using helm

* 3.- Run app or service which wanna expose to outside the world

* 4.- Install cert-manager

* 5.- Create Issuer/ClusterIssuer, Dont forget to change Email

* 6.- Check issuer


## 1.- Installing helm and tillers

Installing Helm sounds easy but actually picky when you don’t have much background information. especially tiller. Follow the steps below to install helm an tiller in your cluster.

**Install helm**

```
curl -s -o helm.tar.gz https://storage.googleapis.com/kubernetes-helm/helm-v2.12.1-linux-amd64.tar.gz \
 && tar -zxvf helm.tar.gz \
 && cp -p -f linux-amd64/helm /usr/local/bin/ \
 && helm init
```

 **Create tiller identity on the k8s cluster**

```
kubectl create serviceaccount — namespace kube-system tiller

kubectl create clusterrolebinding tiller-cluster-rule — clusterrole=cluster-admin — serviceaccount=kube-system:tiller

kubectl patch deploy — namespace kube-system tiller-deploy -p ‘{“spec”:{“template”:{“spec”:{“serviceAccount”:”tiller”}}}}’
```
First line installs helm and tiller. And then creates service account for tiller in kube-system which is a name space


**2.- Install nginx-ingress using helm**

Helm is very useful since you don’t need to wirte the entire yaml file

```
helm install stable/nginx-ingress \
    --namespace kube-system \
    --name nginx-ingress
```
deploying nginx-ingress on kube-system namesapce


**3. Run your application on a default namespace in order to expose it to outside world**

Create a dummy echo application to expose by the nginx-ingress, creating the file echo1.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: echo1
spec:
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: echo1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo1
spec:
  selector:
    matchLabels:
      app: echo1
  replicas: 2
  template:
    metadata:
      labels:
        app: echo1
    spec:
      containers:
      - name: echo1
        image: hashicorp/http-echo
        args:
        - "-text=ECHO1-FOR TESTING"
        ports:
        - containerPort: 5678
```
Then apply it...

```
kubectl apply -f echo1.yaml
```
You should see the following output:

```
Output
service/echo1 created
deployment.apps/echo1 created
```

Now, create a second echo service called echo2 with echo2.yaml file as follow:

```
apiVersion: v1
kind: Service
metadata:
  name: echo2
spec:
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: echo2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo2
spec:
  selector:
    matchLabels:
      app: echo2
  replicas: 1
  template:
    metadata:
      labels:
        app: echo2
    spec:
      containers:
      - name: echo2
        image: hashicorp/http-echo
        args:
        - "-text=echo2"
        ports:
        - containerPort: 5678
```

Then apply it...

```
kubectl apply -f echo2.yaml
```
You should see the following output:

```
Output
service/echo2 created
deployment.apps/echo2 created
```


**4. Cert-manager does all the jobs for maintaining certificates for your SSL certificates. it updates and renews**

```
helm install stable/cert-manager \
    --namespace kube-system \
    --set ingressShim.defaultIssuerName=letsencrypt-prod \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --version v0.5.2
```


**5. Create clusterissuer**

Create clusterissuer.yaml file

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: ???@???.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    http01: {}
```

What you really need to careful at this step is, filling your own email address. And acme-v02 version on server tap. I spent pretty long time following older version tutorials and i dont get the reason qhy with acme-v01 doesn't work

Apply it...

```
kubectl apply -f clusterissuer.yaml
```


**6. Run follow in order to check the status of Clusterissuer**

```
kubectl describe clusterissuer letsencrypt-prod
```

Message and reason are helpful for debuggig and ‘ACMEAccountRegistred’ is right Reason when everything set well

Now you can create a ingress in your echo services namespace

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo-ingress
  annotations:  
    kubernetes.io/ingress.class: nginx
    certmanager.k8s.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - echo1.example.com
    - echo2.example.com
    secretName: letsencrypt-prod
  rules:
  - host: echo1.example.com
    http:
      paths:
      - backend:
          serviceName: echo1
          servicePort: 80
  - host: echo2.example.com
    http:
      paths:
      - backend:
          serviceName: echo2
          servicePort: 80
```
