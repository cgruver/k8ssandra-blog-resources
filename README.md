# K8ssandra Install

Dependency: [https://upstreamwithoutapaddle.com/home-lab/labcli/](https://upstreamwithoutapaddle.com/home-lab/labcli/)

## Set Up For Install

```bash
export K8SSANDRA_WORKDIR=${OKD_LAB_PATH}/k8ssandra-work-dir
mkdir -p ${K8SSANDRA_WORKDIR}/cert-manager-install
mkdir ${K8SSANDRA_WORKDIR}/tmp

git clone https://github.com/cgruver/lab-multi-region.git ${K8SSANDRA_WORKDIR}/lab-multi-region
. ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/versions.sh
```

## Copy Images to Lab Nexus Registry

```bash
podman machine start

labctx cp
podman login -u openshift-mirror ${LOCAL_REGISTRY}

envsubst < ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/images.yaml > ${K8SSANDRA_WORKDIR}/images.yaml
IMAGE_YAML=${K8SSANDRA_WORKDIR}/images.yaml
image_count=$(yq e ".images" ${IMAGE_YAML} | yq e 'length' -)
let image_index=0
while [[ image_index -lt ${image_count} ]]
do
  image_name=$(yq e ".images.[${image_index}].name" ${IMAGE_YAML})
  image_registry=$(yq e ".images.[${image_index}].registry" ${IMAGE_YAML})
  image_version=$(yq e ".images.[${image_index}].version" ${IMAGE_YAML})
  podman pull ${image_registry}/${image_name}:${image_version}
  podman tag ${image_registry}/${image_name}:${image_version} ${LOCAL_REGISTRY}/k8ssandra/${image_name}:${image_version}
  podman push --tls-verify=false ${LOCAL_REGISTRY}/k8ssandra/${image_name}:${image_version}
  image_index=$(( ${image_index} + 1 ))
done
```

## Install Cert Manager

```bash
wget -O ${K8SSANDRA_WORKDIR}/tmp/cert-manager.yaml https://github.com/jetstack/cert-manager/releases/download/${CERT_MGR_VER}/cert-manager.yaml

envsubst < ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/cert-manager-kustomization.yaml > ${K8SSANDRA_WORKDIR}/tmp/kustomization.yaml

kustomize build ${K8SSANDRA_WORKDIR}/tmp > ${K8SSANDRA_WORKDIR}/cert-manager-install.yaml

for i in dc1 dc2 dc3 cp
do
  labctx ${i}
  oc --kubeconfig ${KUBE_INIT_CONFIG} create -f ${K8SSANDRA_WORKDIR}/cert-manager-install.yaml
done

rm ${K8SSANDRA_WORKDIR}/tmp/cert-manager.yaml
```

### Install K8ssandra Operator

```bash
for DEPLOY_TYPE in control-plane data-plane
do
  export DEPLOY_TYPE
  envsubst < ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/k8ssandra-kustomization.yaml > ${K8SSANDRA_WORKDIR}/tmp/kustomization.yaml
  kustomize build ${K8SSANDRA_WORKDIR}/tmp > ${K8SSANDRA_WORKDIR}/k8ssandra-${DEPLOY_TYPE}.yaml
done

labctx cp
oc --kubeconfig ${KUBE_INIT_CONFIG} create -f ${K8SSANDRA_WORKDIR}/k8ssandra-control-plane.yaml
oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator patch role k8ssandra-operator --type=json -p='[{"op": "add", "path": "/rules/-", "value": {"apiGroups": [""],"resources": ["endpoints/restricted"],"verbs": ["create"]} }]'
oc --kubeconfig ${KUBE_INIT_CONFIG} adm policy add-scc-to-user anyuid -z default -n k8ssandra-operator

for i in dc1 dc2 dc3
do
  labctx ${i}
  oc --kubeconfig ${KUBE_INIT_CONFIG} create -f ${K8SSANDRA_WORKDIR}/k8ssandra-data-plane.yaml
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator patch role k8ssandra-operator --type=json -p='[{"op": "add", "path": "/rules/-", "value": {"apiGroups": [""],"resources": ["endpoints/restricted"],"verbs": ["create"]} }]'
  oc --kubeconfig ${KUBE_INIT_CONFIG} adm policy add-scc-to-user anyuid -z default -n k8ssandra-operator
done

for i in dc1 dc2 dc3 cp
do
  labctx ${i}
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment cass-operator-controller-manager --replicas=0
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=0
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator patch configmap cass-operator-manager-config --patch="$(envsubst < ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/cass-config-patch.yaml)"
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment cass-operator-controller-manager --replicas=1
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=1
done
```

## Create ClientConfigs

```bash
mkdir -p ${K8SSANDRA_WORKDIR}/tmp
labctx cp
oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=0

for i in dc1 dc2 dc3
do
  labctx ${i}
  sa_secret=$(oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator get serviceaccount k8ssandra-operator -o yaml | yq e ".secrets" - | grep token | cut -d" " -f3)
  sa_token=$(oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator get secret $sa_secret -o jsonpath='{.data.token}' | base64 -d)
  ca_cert=$(oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator get secret $sa_secret -o jsonpath="{.data['ca\.crt']}")
  cluster=$(oc --kubeconfig ${KUBE_INIT_CONFIG} config view -o jsonpath="{.contexts[0].context.cluster}")
  cluster_addr=$(oc --kubeconfig ${KUBE_INIT_CONFIG} config view -o jsonpath="{.clusters[0].cluster.server}")

  export SECRET_FILE=${K8SSANDRA_WORKDIR}/tmp/kubeconfig

  envsubst < ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/kubeconfig-secret.yaml > ${SECRET_FILE}

  labctx cp

  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator create secret generic ${cluster}-config --from-file="${SECRET_FILE}"

  envsubst < ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/client-config.yaml | oc --kubeconfig ${KUBE_INIT_CONFIG} apply -n k8ssandra-operator -f -

done

labctx cp
oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=1
```

## Deploy Cluster

```bash
labctx cp
envsubst < ${K8SSANDRA_WORKDIR}/lab-multi-region/k8ssandra/k8ssandra-cluster.yaml | oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator apply -f -
```

## Delete Cluster

```bash
labctx cp
oc --kubeconfig ${KUBE_INIT_CONFIG} delete K8ssandraCluster k8ssandra-cluster -n k8ssandra-operator
```

## Delete ClientConfigs

```bash
labctx cp
oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=0
for i in dc1 dc2 dc3
do
  labctx ${i}
  cluster=$(oc --kubeconfig ${KUBE_INIT_CONFIG} config view -o jsonpath="{.contexts[0].context.cluster}")
  labctx cp
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator delete secret ${cluster}-config
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator delete ClientConfig ${cluster}
done
labctx cp
oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator scale deployment k8ssandra-operator --replicas=1
```

## Delete K8ssandra

```bash
labctx cp
oc --kubeconfig ${KUBE_INIT_CONFIG} delete -f ${K8SSANDRA_WORKDIR}/k8ssandra-control-plane.yaml

for i in dc1 dc2 dc3
do
  labctx ${i}
  oc --kubeconfig ${KUBE_INIT_CONFIG} delete -f ${K8SSANDRA_WORKDIR}/k8ssandra-data-plane.yaml
done
```

## Expose Stargate Services:

```bash
for i in dc1 dc2 dc3
do
  labctx ${i}
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator create route edge sg-graphql --service=k8ssandra-cluster-${i}-stargate-service --port=8080
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator create route edge sg-auth --service=k8ssandra-cluster-${i}-stargate-service --port=8081
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator create route edge sg-rest --service=k8ssandra-cluster-${i}-stargate-service --port=8082
done
```

## Expose the Management API:

```bash
for i in dc1 dc2 dc3
do
  labctx ${i}
  oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator create route edge sg-graphql --service=k8ssandra-cluster-${i}-stargate-service --port=8080
done
```

## Connect To the Cluster

```bash
POD_NAME=$(oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator get statefulsets --selector app.kubernetes.io/name=cassandra -o jsonpath='{.items[0].metadata.name}')-0
oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator port-forward ${POD_NAME} 9042

labctx dc1

CLUSTER_INIT_USER=$(oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator get secret k8ssandra-cluster-superuser -o jsonpath="{.data.username}" | base64 -d)
CLUSTER_INIT_PWD=$(oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator get secret k8ssandra-cluster-superuser -o jsonpath="{.data.password}" | base64 -d)

oc --kubeconfig ${KUBE_INIT_CONFIG} -n k8ssandra-operator port-forward svc/k8ssandra-cluster-dc1-stargate-service 9042

cqlsh -u ${CLUSTER_INIT_USER} -p ${CLUSTER_INIT_PWD} -e CREATE ROLE IF NOT EXISTS cajun-navy
```
