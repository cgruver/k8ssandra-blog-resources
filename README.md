# K8ssandra Install

This project contains the configuration resources for a demo of Quarkus with Cassandra and Stargate:

* Blog Post: [Quarkus for Architects who Sometimes Write Code - Being Persistent - Part 01](https://upstreamwithoutapaddle.com/blog%20post/quarkus%20series/2022/10/08/Quarkus-For-Architects-03.html)
* Blog Main Page: [Upstream - Without A Paddle](https://upstreamwithoutapaddle.com/)

## Building K8ssandra for `arm64`

```bash
brew install podman-compose
podman machine start
export PUSH_REGISTRY=quay.io/cgruver0
podman login ${PUSH_REGISTRY}
export BUILDAH_FORMAT=docker
```

```bash
export K8SSANDRA_WORKDIR=${HOME}/okd-lab/quarkus-projects/k8ssandra-work-dir
rm -rf ${K8SSANDRA_WORKDIR}/build
mkdir -p ${K8SSANDRA_WORKDIR}/build
. ${K8SSANDRA_WORKDIR}/k8ssandra-blog-resources/versions.sh
```

### Copy Existing Images to Manifests

```bash
envsubst < ${K8SSANDRA_WORKDIR}/k8ssandra-blog-resources/images.yaml > ${K8SSANDRA_WORKDIR}/images.yaml
IMAGE_YAML=${K8SSANDRA_WORKDIR}/images.yaml
image_count=$(yq e ".images" ${IMAGE_YAML} | yq e 'length' -)

let image_index=0
while [[ image_index -lt ${image_count} ]]
do
  podman manifest rm ${target_registry}/${image_name}:${image_version}
  image_index=$(( ${image_index} + 1 ))
done

let image_index=0
while [[ image_index -lt ${image_count} ]]
do
  image_name=$(yq e ".images.[${image_index}].name" ${IMAGE_YAML})
  source_registry=$(yq e ".images.[${image_index}].source-registry" ${IMAGE_YAML})
  target_registry=$(yq e ".images.[${image_index}].target-registry" ${IMAGE_YAML})
  image_version=$(yq e ".images.[${image_index}].version" ${IMAGE_YAML})
  podman manifest create ${target_registry}/${image_name}:${image_version}
  podman pull --arch=amd64 ${source_registry}/${image_name}:${image_version}
  podman tag ${source_registry}/${image_name}:${image_version} ${image_name}:amd64
  podman manifest add ${target_registry}/${image_name}:${image_version} containers-storage:localhost/${image_name}:amd64
  podman image rm ${source_registry}/${image_name}:${image_version}
  podman pull --arch=arm64 ${source_registry}/${image_name}:${image_version}
  podman tag ${source_registry}/${image_name}:${image_version} ${image_name}:arm64
  podman manifest add ${target_registry}/${image_name}:${image_version} containers-storage:localhost/${image_name}:arm64
  podman image rm ${source_registry}/${image_name}:${image_version}
  image_index=$(( ${image_index} + 1 ))
done
```

### K8ssandra Operator

```bash
git clone https://github.com/cgruver/k8ssandra-operator.git ${K8SSANDRA_WORKDIR}/build/k8ssandra-operator
cd ${K8SSANDRA_WORKDIR}/build/k8ssandra-operator
git checkout ${K8SSANDRA_VER}

IMAGE_TAG_BASE=${PUSH_REGISTRY}/k8ssandra-operator/k8ssandra-operator
podman buildx build --arch=arm64 --build-arg ARCH=arm64 --load -t ${IMAGE_TAG_BASE}:arm64 .
podman manifest add ${IMAGE_TAG_BASE}:${K8SSANDRA_VER} containers-storage:${IMAGE_TAG_BASE}:arm64
```

### Cass Operator

```bash
git clone https://github.com/cgruver/cass-operator.git ${K8SSANDRA_WORKDIR}/build/cass-operator
cd ${K8SSANDRA_WORKDIR}/build/cass-operator
git checkout ${CASS_OPER_VER}
VERSION=$(echo ${CASS_OPER_VER} | cut -d'v' -f2)

IMAGE_TAG_BASE=${PUSH_REGISTRY}/k8ssandra-operator/cass-operator
podman buildx build --arch=arm64 --build-arg VERSION=${VERSION} --build-arg ARCH=arm64 --load -t ${IMAGE_TAG_BASE}:arm64 . 
podman manifest add ${IMAGE_TAG_BASE}:${CASS_OPER_VER} containers-storage:${IMAGE_TAG_BASE}:arm64

IMAGE_TAG_BASE=${PUSH_REGISTRY}/k8ssandra-operator/system-logger
podman buildx build --arch=arm64 --build-arg VERSION=${VERSION} --build-arg TINI_BIN=tini-arm64 --load -t ${IMAGE_TAG_BASE}:arm64  -f logger.Dockerfile . 
podman manifest add ${IMAGE_TAG_BASE}:${CASS_OPER_VER} containers-storage:${IMAGE_TAG_BASE}:arm64
```

## Push Manifests to Registry

```bash
IMAGE_YAML=${K8SSANDRA_WORKDIR}/images.yaml
image_count=$(yq e ".images" ${IMAGE_YAML} | yq e 'length' -)
let image_index=0
while [[ image_index -lt ${image_count} ]]
do
  image_name=$(yq e ".images.[${image_index}].name" ${IMAGE_YAML})
  target_registry=$(yq e ".images.[${image_index}].target-registry" ${IMAGE_YAML})
  image_version=$(yq e ".images.[${image_index}].version" ${IMAGE_YAML})
  podman manifest push --all --tls-verify=false ${target_registry}/${image_name}:${image_version}  ${target_registry}/${image_name}:${image_version}
  image_index=$(( ${image_index} + 1 ))
done
```

## Work In Progress For Additional Images

### Cassandra Reaper

```bash
git clone https://github.com/thelastpickle/cassandra-reaper.git ${K8SSANDRA_WORKDIR}/build/cassandra-reaper

cd ${K8SSANDRA_WORKDIR}/build/cassandra-reaper

mvn clean package
```
