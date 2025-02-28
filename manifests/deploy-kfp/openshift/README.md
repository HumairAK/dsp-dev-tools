This folder contains manifests to help with deploying Kubeflow Pipelines Standalone on Openshift. 

You can deploy standard, or with multi-user auth enabled. 
Before deploying, you will first need to set the remote fork from which to pull the base manifests from. From this we 
apply some patches to get the deployment to work on Openshift. You can inspect these patch files/additions in: 

- manifests/deploy-kfp/openshift/openshift/base
- manifests/deploy-kfp/openshift/openshift/auth


### Configure your repo: 

We'll use the upstream, default branch. 
```bash
# Adjust this to your fork if needed, no trailing slash
REPO=https://github.com/kubeflow/pipelines
BRANCH=master
```

### Deploy base no auth/multi-user

Navigate to the base folder

```bash
git clone https://github.com/opendatahub-io/dsp-dev-tools.git
cd manifests/deploy-kfp/openshift/base
```

Add upstream dependencies and deploy
```bash
./add_resources.sh $REPO $BRANCH
kustomize build . | oc -n kubeflow apply -f -
```

Cache the dependencies for faster builds
```bash
# Speed up the remote pulls by usinzg kustomize localize
kustomize localize . builddir 
kustomize build . | oc -n kubeflow apply -f -
```

### Deploy with auth/multi-user

```bash
git clone https://github.com/opendatahub-io/dsp-dev-tools.git
cd manifests/deploy-kfp/openshift/base
./add_resources.sh $REPO $BRANCH
cd ../auth
kustomize build . | oc -n kubeflow apply -f -
```

### Add your own images: 

To add your own images: 

```bash
# Argo wf controller
SRC=gcr.io/ml-pipeline/workflow-controller
TARGET=quay.io/argoproj/workflow-controller:v3.4.16
kustomize edit set image ${SRC}=${TARGET}

# api-server
SRC=ghcr.io/kubeflow/kfp-api-server
TARGET=quay.io/hukhan/ds-pipelines-api-server:fix_deps_1
kustomize edit set image ${SRC}=${TARGET}

# Persistent Agent
SRC=ghcr.io/kubeflow/kfp-persistence-agent
TARGET=quay.io/hukhan/persistenceagent:fix_deps_1
kustomize edit set image ${SRC}=${TARGET}

# frontend
SRC=ghcr.io/kubeflow/kfp-frontend
TARGET=quay.io/hukhan/ds-pipelines-frontend:2.4.0-release-1
kustomize edit set image ${SRC}=${TARGET}

# scheduledworkflow
SRC=ghcr.io/kubeflow/kfp-scheduled-workflow-controller
TARGET=quay.io/hukhan/ds-pipelines-scheduledworkflow:fix_deps_1
kustomize edit set image ${SRC}=${TARGET}
```

> Note the resource format, read more about it [here](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/remoteBuild.md).

### Add custom driver/launcher images 

These are set as ENV vars to api server, so they need to be changed manually: 

```bash
LAUNCHER_IMAGE=quay.io/hukhan/kfp-launcher:11281
DRIVER_IMAGE=quay.io/hukhan/kfp-driver:11281

# from repo root
cd manifests/deploy-kfp/openshift/base
var=$LAUNCHER_IMAGE yq -i '.spec.template.spec.containers[0].env[1].value = strenv(var)' api-server-patch.yaml
var=$DRIVER_IMAGE yq -i '.spec.template.spec.containers[0].env[0].value = strenv(var)' api-server-patch.yaml
kustomize build . | oc -n kubeflow apply -f -
```
