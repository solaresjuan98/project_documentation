
# Application configuration commands

<i>**This commands are applicable for both environments, development and production**</i>

## Get credentials to development cluster

```bash
gcloud container clusters get-credentials development-gke-cluster --zone us-central1-a
```

## Container Registry secret

```bash

## create secret
kubectl create secret docker-registry container-registry --docker-server=gcr.io --docker-email=finalproject-admin@finalproject-tp.iam.gserviceaccount.com --docker-username=_json_key --docker-password="$(cat <path-to-json-key>)"

## Patch default service account with the gcr-json-key
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'


```

## Workload identity

```bash
# GKE Development Cluster
gcloud container clusters update development-gke-cluster \
    --region=us-central1 \
    --workload-pool=finalproject-tp.svc.id.goog

# node pool
gcloud container node-pools update development-gke-cluster-pool \
    --cluster=development-gke-cluster \
    --workload-metadata=GKE_METADATA

# ===== PRODUCTION ======

# GKE Production Cluster
gcloud container clusters update production-gke-cluster \
    --region=us-central1-a \
    --workload-pool=finalproject-tp-production.svc.id.goog

#
gcloud container node-pools update production-gke-cluster-pool \
    --cluster=production-gke-cluster \
    --workload-metadata=GKE_METADATA

```

## Ingress nginx with exposing metrics

```bash
helm upgrade ingress-nginx ingress-nginx/nginx-ingress \
  --namespace ingress-nginx \
  --set controller.replicaCount=2 \
	--repo https://kubernetes.github.io/ingress-nginx \
	--namespace ingress-nginx \
	--set controller.metrics.enabled=true \
	--set-string controller.podAnnotations."prometheus\.io/scrape"="true" \
	--set-string controller.podAnnotations."prometheus\.io/port"="10254"
  --set controller.service.loadBalancerIP="<IP_ADDRESS>"

helm upgrade ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx \
--set controller.metrics.enabled=true \
--set-string controller.podAnnotations."prometheus\.io/scrape"="true" \
--set-string controller.podAnnotations."prometheus\.io/port"="10254"

helm upgrade -n ingress-nginx

#### PRODUCTION 34.27.179.119
helm upgrade ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--namespace ingress-nginx \
--set controller.metrics.enabled=true \
--set-string controller.podAnnotations."prometheus.io/scrape"="true" \
--set-string controller.podAnnotations."prometheus.io/port"="10254" \
--set controller.service.loadBalancerIP="<IP_ADDRESS>" \
--set controller.metrics.serviceMonitor.enabled=true \ 
--set controller.extraArgs.enable-ssl-passthrough=true \
--set controller.metrics.serviceMonitor.additionalLabels.release="prometheus" \

```

## Cert manager Instalation

```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.10.0

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.10.0 \
  --version v1.10.0 \
  --set webhook.timeoutSeconds=4
```

```

## Argo CD

```bash

# Patch secret (password: argotp)
kubectl -n argocd patch secret argocd-secret -p 
'{"stringData": { "admin.password": "$2y$10$5VFJ5m6V/wo7H2E4y83BXun1wiY529ffADwx9wtIy1r.2Vkp17WK.", "admin.passwordMtime": "$(date +%FT%T%Z)"  } }'

```

## Allowing to service account to have access to GCR

```bash

##
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'

```


## Secrets manager

```bash
${PROJECT_ID} = finalproject-tp

# create secret my-db-password
echo -n "mypassword" | gcloud secrets create my-db-password \
    --project ${PROJECT_ID} \
    --replication-policy automatic \
    --data-file=-

# Verify secrets
gcloud secrets versions access 1 --secret my-db-password

# Grant the GSA the secretAccessor role on the previously created Secret
gcloud secrets add-iam-policy-binding my-db-password \
    --project ${PROJECT_ID} \
    --member="serviceAccount:secret-gsa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# Create a Kubernetes Service Account (KSA)
kubectl create sa --namespace default secret-ksa

# Allow the KSA to impersonate the GSA
gcloud iam service-accounts add-iam-policy-binding \
    secret-gsa@finalproject-tp.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:finalproject-tp.svc.id.goog[test-secrets/secret-ksa]"

# Annotate the KSA
kubectl annotate serviceaccount \
    --namespace test-secrets secret-ksa  \
    iam.gke.io/gcp-service-account=secret-gsa@finalproject-tp.iam.gserviceaccount.com

```

```bash
# Mongodb production string
mongodb+srv://production-admin:finalproject@prodcluster.ft8ukeo.mongodb.net/chatdb

```

## Mongodb Secrets

```bash

# Create secret
echo "mongodb+srv://" | gcloud secrets create mongodb-conn-string \
    --project finalproject-tp \
    --replication-policy automatic \
    --data-file=-

gcloud secrets versions access 1 --secret mongodb-conn-string

# Grant the GSA the secretAccessor role on the previously created Secret
gcloud secrets add-iam-policy-binding mongodb-conn-string \
    --project finalproject-tp \
    --member="serviceAccount:secret-gsa@finalproject-tp.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# Create a Kubernetes Service Account (KSA) in namespace chat-app
kubectl create sa --namespace chat-app secret-ksa

# Allow the KSA to impersonate the GSA
gcloud iam service-accounts add-iam-policy-binding \
    secret-gsa@finalproject-tp.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:finalproject-tp.svc.id.goog[chat-app/secret-ksa]"

# Annotate the KSA
kubectl annotate serviceaccount \
    --namespace chat-app secret-ksa  \
    iam.gke.io/gcp-service-account=secret-gsa@finalproject-tp.iam.gserviceaccount.com

# ================================== PRODUCTION

```

## Mysql / Auth proxy / GKE Connection

## User management

```bash

# if SP does not work
CREATE USER IF NOT EXISTS root@'%' IDENTIFIED BY 'Admin@789**$';
GRANT ALL PRIVILEGES ON *.* TO root @'%' IDENTIFIED BY 'Admin@789**$';
```

```bash

# Auth proxy
cd auth_proxy
./cloud_sql_proxy -instances=finalproject-tp:us-central1:private-instance-dev-57d10b

# login
mysql -u devadmintp -p -h 127.0.0.1

# Backup
mysql --host=127.0.0.1 --user=devadmintp --port=3306 -p BlockBusted < blockbusted.sql

# =========================================================================
kubectl create secret generic mysql-creds  \
  --from-literal=username=devadmintp \
  --from-literal=password='$traineefptp$' \
  --from-literal=database=blockbusted -n movies-app

# ============================= MySQL Secrets =============================

# Database user
# bind policy
gcloud secrets add-iam-policy-binding db-user --member "serviceAccount:external-secrets@finalproject-tp.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

# create generic secret in namespace movies-app
kubectl create secret generic gcpsm-secret-db-user --from-file=secret-access-credentials=key.json -n movies-app

# Database password
# bind policy
gcloud secrets add-iam-policy-binding db-password --member "serviceAccount:external-secrets@finalproject-tp.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

# create generic secret in namespace movies-app
kubectl create secret generic gcpsm-secret-db-password --from-file=secret-access-credentials=key.json -n movies-app

#
kubectl create secret generic db-secret-host --from-literal=db_host="10.14.0.3" -n movies-app

# =================================================================

gcloud secrets add-iam-policy-binding db-host-development --member "serviceAccount:external-secrets@finalproject-tp.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

kubectl create secret generic gcpsm-secret-db-host-development --from-file=secret-access-credentials=key.json -n movies-app

# =================================================================

gcloud secrets add-iam-policy-binding db-name-development --member "serviceAccount:external-secrets@finalproject-tp.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

kubectl create secret generic gcpsm-secret-db-name-development --from-file=secret-access-credentials=key.json -n movies-app

```

## Consuming secrets using the GSM API

```bash

kubectl create ns es
helm install external-secrets external-secrets/external-secrets -n es

# ============================= Mongo db secrets =============================

# create secret
echo mongodb+srv://dev-admin:trainee2022@developmentcluster.7wuqrrn.mongodb.net/chatdb | gcloud secrets create mongodbconnectionstr --data-file=-

# create key.json (if not exists)
gcloud iam service-accounts keys create key.json --iam-account=external-secrets@$project.iam.gserviceaccount.com

# bind policy
gcloud secrets add-iam-policy-binding mongodbconnectionstr --member "serviceAccount:external-secrets@finalproject-tp.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

# create generic secret in namespace es
kubectl create secret generic gcpsm-secret-mongo-conn --from-file=secret-access-credentials=key.json -n es

# ================================== JWT KEY ==================================
echo p@l4Br4zzz3kr3ttttta | gcloud secrets create jwtkey --data-file=-

# bind policy to service account
gcloud secrets add-iam-policy-binding jwtkey --member "serviceAccount:external-secrets@finalproject-tp.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

# Create generic secret in namespace chat-app
kubectl create secret generic gcpsm-secret-jwt-key --from-file=secret-access-credentials=key.json -n chat-app

# =============== PRODUCTION ===============

gcloud secrets add-iam-policy-binding production-mongodb-conn-string --member "serviceAccount:external-secrets@finalproject-tp-production.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding production-jwt-key --member "serviceAccount:external-secrets@finalproject-tp-production.iam.gserviceaccount.com" --role "roles/secretmanager.secretAccessor"

kubectl create secret generic gcpsm-secret-prod --from-file=secret-access-credentials=key.json -n chat-app

```



## Autoscaling

```bash
kubectl autoscale deployment chatappservices --cpu-percent=50 --min=3 --max=10 -n chat-app

kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -n chat-app -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://10.3.251.150; done"

# == Movies

kubectl autoscale deployment moviesappservices --cpu-percent=50 --min=2 --max=10 -n movies-app

kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -n chat-app -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://10.147.250.130; done"

```



