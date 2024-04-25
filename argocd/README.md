#Accessing nginx App on https using ingress in our Kubenetes via argocd

1- Install minikube, enable metallb and ingress 
  minikube start --memory=8182 --disk-size=100GB --cpus=4 --driver=hyperkit 
  minikube addons list
  minikube addons enable metallb
  minikube addons enable ingress

2- Create usefull files and directories
  create argocd-https directory
  create argocd-https/value.yml files
  create directory certmanager-letsencrypt
  create certmanager-letsencrypt/value.yml file
  create certmanager-letsencrypt/Clustertissuer.yml

3- Get cert-manager
  open https://artifacthub.io/ on the browser and search for cert manager. Pick the one manage the company itself cert-manger
  hit the default values link, copy the default value and paste into certmanager-letsencrypt/value.yml file 
  install cert-manager using helm chart
  -kubectl create ns cert-manager (set as default ns)
  -kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
    OR apply
    -helm repo add cert-manager https://charts.jetstack.io
    -helm install cert-manager cert-manager/cert-manager --version 1.14.4 -f certmanager-letsencrypt/value.yml -n cert-manager
    if failed run the following commad
    -helm upgrade cert-manager cert-manager/cert-manager --version 1.14.4 -f certmanager-letsencrypt/value.yml -n cert-manager
  - vim certmanager-letsencrypt/Clustertissuer.yml 
  #(add the following template and edite as well)
##############################################
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dev
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: tsar@net.com
    privateKeySecretRef:
      name: letsencrypt-dev
    solvers:
    - http01:
        ingress:
          class: nginx
#################################################
  -kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.crds.yaml (if needed)
  -kubectl apply -f certmanager-letsencrypt/ClusterIssuer.yml

4- Installing argocd
open https://artifacthub.io/ on the browser and search for argocd
and look for argo-cd with most star.
copy the default value and paste into argocd-https/value.yml
run the following command
  -kubectl create namespace argocd
  -helm repo add argo https://argoproj.github.io/argo-helm
  -helm install tsar-argocd argo/argo-cd --version 6.7.12 -n argocd

edit argocd-https/value.yml file
 -under ## Globally shared configuration (hint, you can search)
 set domain: tsar.argocd.com
 -under # Argo CD server ingress configuration
 enabled: True 
  look for annotations and paste under annotation as 
      annotations: 
        cert-manager.io/cluster-issuer: letsencrypt-dev
        nginx.ingress.kubernetes.io/ssl-passthrough: "true"
        nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
        ingressClassName: "nginx"

-under TLS certificate configuration via cert-manage
set enabled: True
issuer:
 kind: "ClusterIssuer" 
 name: "letsencrypt-dev"
certificateSecret: 
   enabled: True

upgrade argocd installed
helm upgrade tsar-argo argo/argo-cd --version 6.7.12   -n argocd -f argocd-https/value.yml 
kubectl get ingress -n argocd 
copy hosts and address (tsar.argocd.com   192.168.67.6)
sudo vim /etc/hosts (Passord required) and add the following
 192.168.67.6  tsar.argocd.com 

5- Accessing from the browser with
https://tsar.argocd.com

go back in the terminal and run the following command to get argodc password. username is admin

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d


