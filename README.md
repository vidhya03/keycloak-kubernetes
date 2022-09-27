# keycloak-kubernetes

This tutorial shows how to deploy a Keycloak cluster to Kubernetes.

All the deployment steps together with a quick cluster test are available on YouTube: [Deploying Keycloak cluster to Kubernetes](https://www.youtube.com/watch?v=g8LVIr8KKSA&list=PLPZal7ksxNs0mgScrJxrggEayV-TPZ9sA&index=1).

I also provide a cloud native demo app which contains:

- React front-end application authenticating with Keycloak using official Keycloak JavaScript adapter [lukaszbudnik/hotel-spa](https://github.com/lukaszbudnik/hotel-spa)
- haproxy acting as an authentication & authorization gateway implemented by [lukaszbudnik/haproxy-auth-gateway](https://github.com/lukaszbudnik/haproxy-auth-gateway)
- mock backend microservices implemented by [lukaszbudnik/yosoy](https://github.com/lukaszbudnik/yosoy)
- ready-to-import Keycloak realm with predefined client, roles, and test users

The deployment and a walkthrough of the demo apps is also available on YouTube: [Keycloak - Distributed apps end-to-end demo](https://www.youtube.com/watch?v=J42sR1t7Vt0&list=PLPZal7ksxNs0mgScrJxrggEayV-TPZ9sA&index=8).

If you want to learn more about Keycloak see [Building cloud native apps: Identity and Access Management](https://dev.to/lukaszbudnik/building-cloud-native-apps-identity-and-access-management-1e5m).

# Deploy Keycloak cluster locally

See [Prerequisites](PREREQUISITES.md) to make sure you have Kubernetes Dashboard, nginx-ingress, and bitnami helm repo configured.

If you have all the prerequisites, then you are just a few commands from running your first Keycloak cluster on Kubernetes:

```bash
# create dedicated namespace for our deployments
kubectl create ns hotel
# deploy PostgreSQL cluster
helm install -n hotel keycloak-db bitnami/postgresql-ha
# deploy Keycloak cluster
kubectl apply -n hotel -f keycloak.yaml
# create HTTPS ingress for Keycloak # Or create without -subj , and enter manually if error raise 
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout auth-tls.key -out auth-tls.crt -subj "/CN=auth.localtest.me/O=hotel"

kubectl create secret -n hotel tls auth-tls-secret --key auth-tls.key --cert auth-tls.crt
kubectl apply -n hotel -f keycloak-ingress.yaml
```

Keycloak is now available at: https://auth.localtest.me.

# Install demo apps

I have provided a sample Hotel SPA application which authenticates with Keycloak using official Keycloak JavaScript adapter, obtains JSON Web Token, and then uses it to talk to the protected services using HTTP Authorization Bearer mechanism.

The architecture diagram looks like this:

![Keycloak demo apps](keycloak-demo-apps.png?raw=true)

## Import ready-to-use Keycloak realm

Keycloak offers partial exports of clients and users/roles from the admin console. There is also a low level import/export functionality which can export complete realm (together with cryptographic keys). I already setup such realm with clients, roles, and test users. I also exported RSA public key in pem format.

After this step a hotel realm will be setup with the following 2 users:

- `julio` with password `julio` and role `camarero`
- `angela` with password `angela` and roles `camarero`, `doncella`, and `cocinera`

There will be also `react` client setup with the following settings:

- `public` access type
- `realm roles` mapper added
- application URLs set to https://lukaszbudnik.github.io/hotel-spa/ (publicly accessible single-page application)

```bash
# find first keycloak pod
POD_NAME=$(kubectl get pods -n hotel -l app=keycloak --no-headers -o custom-columns=":metadata.name" | head -1)
# copy realm to the pod
kubectl cp -n hotel demo/keycloak/hotel_realm.json $POD_NAME:/tmp/hotel_realm.json
# import realm
kubectl exec -n hotel $POD_NAME -- \
/opt/jboss/keycloak/bin/standalone.sh \
-Djboss.socket.binding.port-offset=100 \
-Dkeycloak.migration.action=import \
-Dkeycloak.migration.provider=singleFile \
-Dkeycloak.migration.file=/tmp/hotel_realm.json
```

As you can see the last command starts Keycloak. That's how import/export actually works. Yes, I know... why there is no option for import/export and then exit? Look for the following messages in the log:

```
06:57:21,510 INFO  [org.keycloak.exportimport.singlefile.SingleFileImportProvider] (ServerService Thread Pool -- 64) Full importing from file /tmp/hotel_realm.json
06:57:25,264 INFO  [org.keycloak.exportimport.util.ImportUtils] (ServerService Thread Pool -- 64) Realm 'hotel' imported
06:57:25,332 INFO  [org.keycloak.services] (ServerService Thread Pool -- 64) KC-SERVICES0032: Import finished successfully
```

And then press [CTRL]+[C] to exit.

More on import/export functionality can be found in Keycloak documentation: https://www.keycloak.org/docs/latest/server_admin/#_export_import.

## Deploy mock backend services

Mock backend services are powered by [lukaszbudnik/yosoy](https://github.com/lukaszbudnik/yosoy). If you haven't used any mocking backend services I highly recommend using yosoy. yosoy is a perfect tool for mocking distributed systems.

There are 3 mock services available:

- doncella
- cocinera
- camarero

They are all deployed as headless services and there is no way end-user can access them directly. All HTTP requests must come through API gateway.

```bash
kubectl apply -n hotel -f demo/apps/hotel.yaml
```

## Deploy haproxy-auth-gateway

Authentication and authorization is implemented by haproxy-auth-gateway project.

haproxy-auth-gateway is drop-in transparent authentication and authorization gateway for cloud native apps. For more information please see [lukaszbudnik/haproxy-auth-gateway](https://github.com/lukaszbudnik/haproxy-auth-gateway).

In this demo I implemented the following ACLs:

- `/camarero` is only allowed if JWT token is valid and contains `camarero` role
- `/doncella` is only allowed if JWT token is valid and contains `doncella` role
- `/cocinera` is only allowed if JWT token is valid and contains `cocinera` role

```bash
# create configmaps
kubectl create configmap -n hotel haproxy-auth-gateway-iss-cert --from-file=demo/haproxy-auth-gateway/config/hotel.pem
kubectl create configmap -n hotel haproxy-auth-gateway-haproxy-cfg --from-file=demo/haproxy-auth-gateway/config/haproxy.cfg
# deploy the gateway
kubectl apply -n hotel -f demo/haproxy-auth-gateway/gateway.yaml
# create HTTPS ingress for the gateway
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout api-tls.key -out api-tls.crt -subj "/CN=api.localtest.me/O=hotel"
kubectl create secret -n hotel tls api-tls-secret --key api-tls.key --cert api-tls.crt
kubectl apply -n hotel -f demo/haproxy-auth-gateway/gateway-ingress.yaml
```

The API gateway is now available at: https://api.localtest.me.

## Public hotel-spa app

The front-end application is [lukaszbudnik/hotel-spa](https://github.com/lukaszbudnik/hotel-spa) which is a React single-page application hosted on GitHub pages: https://lukaszbudnik.github.io/hotel-spa/.

Think of GitHub pages as our CDN.

It uses the following keylocak.json configuration:

```json
{
  "realm": "hotel",
  "auth-server-url": "https://auth.localtest.me/auth/",
  "ssl-required": "external",
  "resource": "react",
  "public-client": true,
  "confidential-port": 0
}
```

If you followed all the steps in the above tutorial the app is ready to be used: https://lukaszbudnik.github.io/hotel-spa!

# Deploy Keycloak cluster to AWS EKS

You can follow up below instructions together with the following YouTube video: [Deploying Keycloak cluster to AWS EKS](https://youtu.be/BuNZ7bjbzOQ).

The following env variables have to be set:

```bash
export AWS_REGION=us-east-2
export CLUSTER_NAME=keycloak
# using "auth.myproduct.example.org" as an example, I assume the following:
# 1. AWS Route53 hostedzone "myproduct.example.org" exists - external-dns addon will autodiscover it based on DNS name and add "auth.myproduct.example.org" record to it
# 2. AWS Certificate Manager certificate "auth.myproduct.example.org" exists - aws-load-balancer-controller will autodiscover this certificate based on DNS name and use it for HTTPS listener
export CLUSTER_DNS_NAME=auth.myproduct.example.org
```

I use eksctl tool to simplify all the EKS cluster management: https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html.

Provision the cluster:

```bash
# 1. create cluster
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --version 1.20 \
  --with-oidc \
  --alb-ingress-access \
  --external-dns-access \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 5 \
  --managed

# 2. install aws-load-balancer-controller
helm repo add eks https://aws.github.io/eks-charts
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

export VPC_ID=$(eksctl get cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --output json \
  | jq -r '.[0].ResourcesVpcConfig.VpcId')

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=$CLUSTER_NAME \
  --set region=$AWS_REGION \
  --set vpcId=$VPC_ID \
  --set serviceAccount.name=aws-load-balancer-controller \
  -n kube-system

# 3. install external-dns addon
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install external-dns bitnami/external-dns \
  --set provider=aws \
  -n kube-system
```

Now deploy Keycloak cluster, the only bit that is different from local Keycloak cluster deployment is the ingress:

```bash
# create dedicated namespace for our deployments
kubectl create ns hotel
# deploy PostgreSQL cluster (we could also use AWS RDS PostgreSQL but keeping things simple)
helm install -n hotel keycloak-db bitnami/postgresql-ha
# deploy Keycloak cluster
kubectl apply -n hotel -f keycloak.yaml
# deploy AWS ALB ingress
# substitude PLACEHOLDER_HOSTNAME with target domain name
sed "s/PLACEHOLDER_HOSTNAME/$CLUSTER_DNS_NAME/" keycloak-ingress-eks-placeholder.yaml > keycloak-ingress-eks.yaml
# deploy the ingress
kubectl apply -n hotel -f keycloak-ingress-eks.yaml
```

Wait a minute untill ALB is provisioned and Route53 records are created, then open Keycloak:

```bash
open https://$CLUSTER_DNS_NAME
```

## AWS IAM Mappings

If you use different IAM user/role for eksctl and AWS console you need to add IAM mappings:

```bash
# ARN of either IAM role or IAM user
export AWS_IAM_ARN=$(aws sts get-caller-identity | jq -r '.Arn')

eksctl create iamidentitymapping \
  --cluster $CLUSTER_NAME \
  --arn $AWS_IAM_ARN \
  --username admin \
  --group system:masters
```
