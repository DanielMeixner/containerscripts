# Create ingress with GitHub oauth authentication
See also https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/auth/oauth-external-auth
and https://github.com/denniszielke/container_demos/blob/master/oAuth2Proxy.md

## Prep
- Create AKS Cluster 
- Create GitHub app https://github.com/settings/applications/new 
  - Make sure you use http**s** in your Homepage URL and Callback URL. (Not http!)
  - Callback URL is your homepage URL with /oauth2
  - Your homepage URL is the URL you'll be using as ingress URL  

## Script
```
DNSNAME="dmxauth"
CLUSTERNAME="dmxauth"
RG="dmxauthrg"
LOCATION="eastus"
FQDN=$DNSNAME".eastus.cloudapp.azure.com"
CLUSTERRG="MC_${RG}_${CLUSTERNAME}_${LOCATION}"

# GITHub Secrets (taken from your Github app)
COOKIE=""
CLIENTID=""
CLIENTSECRET=""

## Create loadBalancerIP
az network public-ip create --resource-group $CLUSTERRG --name myAKSPublicIP --allocation-method static
```
Now check the IP you got and use this IP in the follwing section.

 ```
# Set your IP
IP=YYY.AAA.CCC.DDD

# Follow aks tutorial 
##Install Service Account
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF

## init helm 
helm init --service-account tiller

## Create ing controller into default ns
helm install stable/nginx-ingress --namespace default --set controller.service.loadBalancerIP=$IP

# Get the resource-id of the public ip
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

# Update public ip address with DNS name
az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME

# Cert 
## Install Cert manager
helm install stable/cert-manager --set ingressShim.defaultIssuerName=letsencrypt-prod --set ingressShim.defaultIssuerKind=ClusterIssuer

## create cluster ClusterIssuerapiVersion: certmanager.k8s.io/v1alpha1

cat <<EOF | kubectl create -f -
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@mail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    http01: {}
EOF

## Create cert object

cat <<EOF | kubectl create -f -
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: dmxauth-tls-secret
spec:
  secretName: dmxauth-tls-secret
  dnsNames:
  - $FQDN
  acme:
    config:
    - http01:
        ingressClass: nginx
      domains:
      - $FQDN
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
EOF


## Demo Apps
helm install azure-samples/aks-helloworld
helm install azure-samples/aks-helloworld --set title="AKS Ingress Demo" --set serviceName="ingress-demo"
kubectl run nginx --image nginx --port=80
kubectl expose deployment nginx --type=ClusterIP

# Ingress
cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    certmanager.k8s.io/cluster-issuer: letsencrypt-prod    
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - $FQDN
    secretName: dmxauth-tls-secret
  rules:
  - host: $FQDN
    http:
      paths:
      - path: /
        backend:
          serviceName: aks-helloworld
          servicePort: 80
      - path: /hello-world-two
        backend:
          serviceName: ingress-demo
          servicePort: 80
      - path: /nginx
        backend:
          serviceName: nginx
          servicePort: 80
EOF
```
If you access your website now, your cert is valid and you get the right content (your ingress works) However you don't have oauth.

So let's configure oauth.

```
cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    k8s-app: oauth2-proxy
  name: oauth2-proxy
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: oauth2-proxy
  template:
    metadata:
      labels:
        k8s-app: oauth2-proxy
    spec:
      containers:
      - args:
        - --provider=github
        - --email-domain=*
        - --upstream=file:///dev/null
        - --http-address=0.0.0.0:4180
        # Register a new application
        # https://github.com/settings/applications/new
        env:
        - name: OAUTH2_PROXY_CLIENT_ID
          value: $CLIENTID
        - name: OAUTH2_PROXY_CLIENT_SECRET
          value: $CLIENTSECRET
        # docker run -ti --rm python:3-alpine python -c 'import secrets,base64; print(base64.b64encode(base64.b64encode(secrets.token_bytes(16))));'
        - name: OAUTH2_PROXY_COOKIE_SECRET
          value: $COOKIE
        image: docker.io/colemickens/oauth2_proxy:latest
        imagePullPolicy: Always
        name: oauth2-proxy
        ports:
        - containerPort: 4180
          protocol: TCP
EOF

## configure the proxy service

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: oauth2-proxy
  name: oauth2-proxy
  namespace: default
spec:
  ports:
  - name: http
    port: 4180
    protocol: TCP
    targetPort: 4180
  selector:
    k8s-app: oauth2-proxy
EOF

## add the ingress for oauth

cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: oauth2-proxy
  namespace: default
spec:
  rules:
  - host: $FQDN
    http:
      paths:
      - backend:
          serviceName: oauth2-proxy
          servicePort: 4180
        path: /oauth2
  tls:
  - hosts:
    - $FQDN
    secretName: dmxauth-tls-secret
EOF
```

Now you have everything in place. Modify your app-ingress with the following annotations.

```
# nginx.ingress.kubernetes.io/auth-signin: https://$host/oauth2/start?rd=$request_uri
# nginx.ingress.kubernetes.io/auth-url: https://$host/oauth2/auth
kubectl edit ingress app-ingress
```

