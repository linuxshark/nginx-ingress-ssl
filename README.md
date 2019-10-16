# **Nginx Ingress controller with LetsEncrypt automatic SSL cert**

## steps

* 1.- Install helm and tillers

* 2.- Install nginx-ingress using helm

* 3.- Run app or service which wanna expose to outside the world

* 4.- Install cert-manager

* 5.- Create Issuer/ClusterIssuer, Dont forget to change Email

* 6.- Check issuer


## Installing helm and tillers

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

**Install nginx-ingress using helm**

2. Helm is very useful since you don’t need to wirte the entire yaml file
helm install stable/nginx-ingress \
    --namespace kube-system \
    --name nginx-ingress
deploying nginx-ingress on kube-system namesapce
3. Run your application on a default namespace in order to expose it to outside world
4. Cert-manager does all the jobs for maintaining certificates for your SSL certificates. it updates and renews
helm install stable/cert-manager \
    --namespace kube-system \
    --set ingressShim.defaultIssuerName=letsencrypt-prod \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --version v0.5.2
the latest version is missing something i don’t understand. it kept asking me to make secretes so I decided to use “post-newest version” the latest version now is 0.6.0 which is right after 0.5.2
(latest version doesn’t contain CRDs. So it will be better version down)
5. Create clusterissuer
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    email: ???@???.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    http01: {}
What you need to careful at this step is, filling your own email address. And acme-v02 version on server tap. I spent pretty long time following older version tutorials
6. Run follow in order to check the status of Clusterissuer
kubectl describe clusterissuer letsencrypt-prod
Message and reason are helpful for debuggig and ‘ACMEAccountRegistred’ is right Reason when everything set well