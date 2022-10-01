# Configure ingress
enable the minikube ingress addon by running

```
minikube addons enable ingress

```
You can confirm by running the 

```
kubectl get pods -n ingress-nginx
```


## nginx-ingress

Make sure you have nginx ingress installed too:

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx -n ingress-nginx --create-namespace ingress-nginx/ingress-nginx
```


kubectl create ns amma
kubectl config set-context --current --namespace=amma
# deploy PostgreSQL cluster
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install -n amma keycloak-db bitnami/postgresql-ha

# config map for realm
kubectl apply -n amma -f keycloak-relam-config-map.yaml
# deploy Keycloak cluster
kubectl apply -n amma -f keycloak-19.0.yaml


# create HTTPS ingress for Keycloak  and enter manually 
<!-- Country Name (2 letter code) [AU]:.
State or Province Name (full name) [Some-State]:.
Locality Name (eg, city) []:.
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Labkit
Organizational Unit Name (eg, section) []:.
Common Name (e.g. server FQDN or YOUR name) []:auth.labkit.in
Email Address []:vidhya@labkit.in -->

 openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout auth-labkit-tls.key -out auth-labkit-tls.crt

# Create secret for https
 kubectl create secret -n amma tls auth-labkit-tls-secret --key auth-labkit-tls.key --cert auth-labkit-tls.crt

# Ingress for keycloak to access in auth.labkit.in
  kubectl apply -n amma -f keycloak-ingress-labkit.yaml

# Ingress for http
kubectl apply -n amma -f keycloak-ingress-labkit-http.yaml















--------------------------

helm install ingress-nginx -n ingress-nginx --create-namespace ingress-nginx/ingress-nginx
NAME: ingress-nginx
LAST DEPLOYED: Sat Sep 17 18:09:09 2022
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
