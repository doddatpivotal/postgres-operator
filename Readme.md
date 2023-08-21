# Tanzu Sql - Posgres

[Docs](https://docs.vmware.com/en/VMware-Tanzu-SQL-with-Postgres-for-Kubernetes/1.7/tanzu-postgres-k8s/GUID-install-operator.html)

## Environment Value Pre-reqs

Copy the `values-REDACTED.yaml` file to `local-config/` directory and complete with your environment information.  Then export reference to file.

```bash
cp values-REDACTED.yaml local-config/values.yaml
# manually edit the environment value
PARAMS_YAML=local-config/values.yaml
```

## Cluster Pre-req

- Use a fresh TKGm cluster that does not have cert-manager from the extensions deployed
- Ensure 2 worker nodes with 3 vCPU or higher (each pg node requires .8 CPU)

## Cert-Manager Pre-req

```bash
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager  --version v1.0.2 --set installCRDs=true
```

## Setup some environment variables

```bash
VERSION=$(yq e .postgresOperator.version $PARAMS_YAML)
REGISTRY=$(yq e .postgresOperator.registry $PARAMS_YAML)
PROJECT=$(yq e .postgresOperator.project $PARAMS_YAML)
REGISTRY_USERNAME=$(yq e .postgresOperator.username $PARAMS_YAML)
REGISTRY_PASSWORD=$(yq e .postgresOperator.password $PARAMS_YAML)
TANZU_NET_USERNAME=$(yq e .tanzuNet.username $PARAMS_YAML)
TANZU_NET_PASSWORD=$(yq e .tanzuNet.password $PARAMS_YAML)
```

## Download & Extract the operator chart

```bash
helm registry login registry.tanzu.vmware.com \
       --username=$TANZU_NET_USERNAME \
       --password=$TANZU_NET_PASSWORD


helm pull oci://registry.tanzu.vmware.com/tanzu-sql-postgres/postgres-operator-chart --version v$VERSION --untar --untardir /tmp

```

# Install the Operator

```bash
kubectl create secret docker-registry regsecret \
    --docker-server=https://registry.tanzu.vmware.com/ \
    --docker-username=$TANZU_NET_USERNAME \
    --docker-password=$TANZU_NET_PASSWORD \
    --namespace service-instances

helm install my-postgres-operator /tmp/postgres-operator/ --wait  \
    --namespace service-instances

kubectl get all --selector app=postgres-operator -n service-instances
```

## Go ahead and follow the demo script

[Demo Script](demo.md)
