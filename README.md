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

$ docker image ls
REPOSITORY                                                                    TAG       IMAGE ID       CREATED         SIZE
pega-docker.downloads.pega.com/platform/installer                             8.23.1    600173d93113   2 weeks ago     3.87GB
pega-docker.downloads.pega.com/platform/pega                                  8.23.1    2cbc4406e8f9   2 weeks ago     442MB
pega-docker.downloads.pega.com/platform/clustering-service                    1.3.15    afdcbebb442d   4 weeks ago     995MB
pega-docker.downloads.pega.com/platform-services/search-n-reporting-service   1.29.1    53b3ce06d221   5 weeks ago     267MB
pega-docker.downloads.pega.com/constellation-appstatic-service/docker-image   1.11.0    b493aecf339f   8 weeks ago     179MB
pega-docker.downloads.pega.com/platform/clustering-service-kubectl            1.3.14    1f3ec2d9b063   3 months ago    168MB
pega-docker.downloads.pega.com/constellation-messaging/docker-image           5.4.0     d4087ef3a923   4 months ago    213MB

9) Add Helm repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add elastic https://helm.elastic.co
helm repo add pega https://pegasystems.github.io/pega-helm-charts

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

13) Install Kafka
helm install kafka oci://registry-1.docker.io/bitnamicharts/kafka --set listeners.client.protocol=PLAINTEXT --set listeners.controller.protocol=PLAINTEXT --set listeners.interbroker.protocol=PLAINTEXT --set listeners.external.protocol=PLAINTEXT

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka.default.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092
    kafka-controller-1.kafka-controller-headless.default.svc.cluster.local:9092
    kafka-controller-2.kafka-controller-headless.default.svc.cluster.local:9092


14) Install PEGA
helm install mypega pega/pega --values pega-min.yaml --set global.actions.execute=install-deploy

15) Get PEGA service port
minikube service pega-minikube

16) Login
administrator@pega.com
pegapega


michaelchan99@gmail.com
password:
5N,AU'"ewe


michaelchan99@hotmail.com
password:
s(ZX-XgJ8c