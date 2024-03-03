###
This file used to deploy PEGA into minikube
###

1) Download and Install Minikube
https://minikube.sigs.k8s.io/docs/start/

2) Install Helm
https://helm.sh/docs/intro/install/

winget install --id=Helm.Helm  -e

3) Install kubectl
https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

winget install -e --id Kubernetes.kubectl

4) Config Minikube

minikube config set cpus 8
minikube config set memory 22g
minikube config set disk-size 150g

minikube addons enable storage-provisioner
minikube addons enable dashboard
minikube addons enable default-storageclass

5) Start Minikube

minikkube start

6) Start Minikube Registry

minikube addons enable metrics-server
minikube addons enable registry

7) Reset minikube docker user password

minikube ssh
sudo passwd docker
## Copy all Docker image from host to minikube
scp *.tar docker@$(minikube ip):/home/docker

8) Load Image to Minikube

minikube ssh
## Load Image into Docker daemon
docker load < clustering-service-kubectl.1.3.14.tar
docker load < clustering-service.1.3.15.tar
docker load < docker-image.1.11.0.tar
docker load < docker-image.5.4.0.tar
docker load < installer.8.23.1.tar
docker load < pega.8.23.0.tar
docker load < search-n-reporting-service.1.29.1.tar
## check if image load
docker image ls

9) Add Helm repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add elastic https://helm.elastic.co

10) Add Persistent Volume and Persistent Volume Claim, and Install PostgreSQL 
cd pega\pega-minikube\single-minikube\postgreSQL
kubectl apply -f .\postgres-pv.yaml
helm install postgres-helm bitnami/postgresql --version 14.2.3 --set primary.persistence.existingClaim=postgresql-pv-claim --set volumePermissions.enabled=true --set auth.postgresPassword=postgres --set image.tag=14.11.0-debian-12-r6 --set global.postgresql.auth.username=pega --set global.postgresql.auth.password=pega --set global.postgresql.auth.database=pega
## ** Wait for Image download
##
## Check with Minikube dashboard

11) Install pgAdmin
winget install -e --id PostgreSQL.pgAdmin
##
## Forward PostgreSQL port
kubectl port-forward --namespace default svc/postgres-helm-postgresql 5432:5432

## Check PostgreSQL connection
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgres-helm-postgresql -o jsonpath="{.data.password}" | base64 -d)
kubectl port-forward --namespace default svc/postgres-helm-postgresql 5432:5432 & PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U pega -d pega -p 5432
##
## Setup pgAdmin with username and password
##
## Host: localhost
## Username: pega
## Password: pega
## Database: pega

12) Install Elastic Search
## ElasticSearch docker image version can be check on https://www.docker.elastic.co/r/elasticsearch
cd .\pega\pega-minikube\single-minikube\elasticsearch
helm install es-minikube elastic/elasticsearch -f values.yaml --set imageTag=8.10.3
kubectl get pods --namespace=default -l app=elasticsearch-master -w
kubectl get services -n default

## Test if ElasticSearch is working
## Windows do not have base64, can use https://www.base64decode.org/
kubectl get secrets --namespace=default elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
## Test with build-in utility
helm --namespace=default test es-minikube
## Check with port forword
kubectl port-forward svc/elasticsearch-master 9200:9200
## Open brower and https://localhost:9200/
## Login with username: elastic , password: from above base64 decode
## It should display elastic search verison number

13) Install PEGA