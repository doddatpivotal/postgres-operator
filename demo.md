# Demo

## Prereqs

```bash
REGISTRY=$(yq e .postgresOperator.registry $PARAMS_YAML)
PROJECT=spring-music
VERSION=$(yq e .postgresOperator.version $PARAMS_YAML)

kubectl create ns postgres
docker build /tmp/postgres-for-kubernetes-v${VERSION}/sample-app/ -t ${REGISTRY}/${PROJECT}/spring-music:latest 
docker push ${REGISTRY}/${PROJECT}/spring-music:latest
```

## Demo

```bash
# let's checkubectlout the operator
kubens default
kubectl get all

# see that images are pulled from private registry
kubectl describe deployment/postgres-operator

# let's create a database
kubens postgres
bat pg-instance-example-1.yaml
kubectl apply -f pg-instance-example-1.yaml
kubectl get postgres

# Repeat following until all pods are ready.
kubectl get all
kubectl get secret

# Let's connect to the database
kubectl exec -it pg-instance-example-0 -- /bin/bash
psql pg-instance-example -c '\l'
psql pg-instance-example -c '\dt'
exit

# let's bind an application
bat /tmp/postgres-for-kubernetes-v${VERSION}/sample-app/spring-music.yaml

ytt -f local-config/values.yaml -f image-overlay.yaml -f /tmp/postgres-for-kubernetes-v${VERSION}/sample-app/spring-music.yaml | kubectl apply -f -

kubectl get all

# go visit the application

# how about adding backup options
bat s3-secret-example.yaml
ytt -f s3-secret-example.yaml -f local-config/values.yaml | kubectl apply -f -
bat pg-instance-example-2.yaml
kubectl apply -f pg-instance-example-2.yaml
kubectl get postgres
kubectl get po

kubectl exec -it pg-instance-example-0 -- bash -c 'pgbackrest stanza-create --stanza=postgres-pg-instance-example'
kubectl exec -it pg-instance-example-0 -- bash -c 'pgbackrest backup --stanza=postgres-pg-instance-example'

# grab the current timestamp so that we know what point to restore to.
CUR_DATE=$(date +"%Y-%m-%d %H:%M:%S")

# check the current contents of the db table
kubectl exec -it pg-instance-example-0 -- bash -c "psql pg-instance-example -c 'select * from album;'"

# let's delete some data. Use UI to delete data.  Now see that it is missing from table
kubectl exec -it pg-instance-example-0 -- bash -c "psql pg-instance-example -c 'select * from album;'"

# let's perform the restore
# Get the pg_auto_failover child process pid:
PG_AUTO_PID=$(kubectl exec -t pg-instance-example-0 -- ps -ef | grep "pg_autoctl: start/stop postgres" | grep -v grep | awk '{print $2}')
echo $PG_AUTO_PID

# Stop the pg_auto_failover child process:
kubectl exec -t pg-instance-example-0 -- kill -STOP $PG_AUTO_PID

# Stop the Postgres instance:
kubectl exec -t pg-instance-example-0 -- pg_ctl stop

# Perform the restore to the timestamp
kubectl exec -t pg-instance-example-0 -- bash -c "pgbackrest restore --stanza=postgres-pg-instance-example --delta --type=time '--target=$CUR_DATE' --target-action=promote"

# Restart the pg_auto_failover child process (which will bring a Postgres process backubectlup):
kubectl exec -t pg-instance-example-0 -- kill -CONT $PG_AUTO_PID

kubectl get po

# to to site and see if you see the data
kubectl exec -it pg-instance-example-0 -- bash -c "psql pg-instance-example -c 'select * from album;'"

# How about HA
bat pg-instance-example-3.yaml
kubectl apply -f pg-instance-example-3.yaml
kubectl get postgres
kubectl get po

# Now let's watch and see what happens if we kill the primary node in the cluster
kubectl exec -ti pod/pg-instance-example-0 -- pg_autoctl show state

# update below to match secondary
# crazy statement greps for line that includes secondary, then grabs the full node name, then splits on : to retrieve node wihtout port, then gets first item
SECONDARY_POD=$(kubectl exec -ti pod/pg-instance-example-0 -- pg_autoctl show state | grep secondary | awk '{print $5}' | cut -d ':' -f 1 | cut -d '.' -f 1)
watch -n 3 kubectl exec -ti pod/$SECONDARY_POD -- pg_autoctl show state

# open new screen
watch kubectl get po

# open new screen, then update below to match primary
PRIMARY_POD=$(kubectl exec -ti pod/pg-instance-example-0 -- pg_autoctl show state | grep primary | awk '{print $5}' | cut -d ':' -f 1 | cut -d '.' -f 1)
kubectl delete po $PRIMARY_POD

# close both screens

# Why Not Official OSS, Docker Image, Helm

# go to helm, comment about helm., we like helm.  we use helm to deploy our operator.  but helm is not a tool for managing fleets of specialized components

# How about DB and fleets of databases, as a data administrator, I want to know all my postgres dbs out there.

kubectl get postgres -A
```