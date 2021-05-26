# Tanzu Sql - Posgres

[Docs](https://postgres-kubernetes.docs.pivotal.io/1-1/installing.html)

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
```

## Download & Extract

```bash
pivnet download-product-files --product-slug='tanzu-sql-postgres' --release-version='1.1.0' --product-file-id=893946 --download-dir /tmp
tar -zxvf /tmp/postgres-for-kubernetes-v${VERSION}.tar.gz -C /tmp
TEMP_PATH=/tmp/postgres-for-kubernetes-v${VERSION}
```

# Push to Harbor

```bash
INSTANCE_IMAGE_NAME="${REGISTRY}/${PROJECT}/postgres-instance:$(cat ${TEMP_PATH}/images/postgres-instance-tag)"
docker tag $(cat ${TEMP_PATH}/images/postgres-instance-id) ${INSTANCE_IMAGE_NAME}
docker push ${INSTANCE_IMAGE_NAME}

OPERATOR_IMAGE_NAME="${REGISTRY}/${PROJECT}/postgres-operator:$(cat ${TEMP_PATH}/images/postgres-operator-tag)"
docker tag $(cat ${TEMP_PATH}/images/postgres-operator-id) ${OPERATOR_IMAGE_NAME}
docker push ${OPERATOR_IMAGE_NAME}
```

# Create Registry Secret

```bash
kubectl create secret docker-registry regsecret \
    --docker-server=${REGISTRY} \
    --docker-username=${REGISTRY_USERNAME} \
    --docker-password=${REGISTRY_PASSWORD}
```

## Deploy the operator

```bash
helm install postgres-operator ${TEMP_PATH}/operator/ -f values.yaml
```

## Go ahead and follow the demo script

[Demo Script](demo.md)
