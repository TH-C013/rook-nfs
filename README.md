# Get Rook-Ceph Helm Chart

**_* Add chart repository locally -Setup Once- *_**
helm repo add rook-release https://charts.rook.io/release

**_* Download a chart specified version from helm repository and unpacks it in local directory *_**
helm pull rook-release/rook-ceph --version 1.8.5 --untar

**\_\__ OPTION-1 _\_\_**

_IMPORTANT_

**Use the steps below for the configuration instead**

# Install an NFS Server using the NFS Quick Start guide below - Note this guide is for Rook V1.7

https://rook.io/docs/nfs/v1.7/quickstart.html

# Deploy NFS Operator

First deploy the Rook NFS operator using the following commands:
Clone the git repo and change directory to `rook/cluster/examples/kubernetes/nfs`

```
    git clone --single-branch --branch v1.7.3 https://github.com/rook/nfs.git
    cd rook/cluster/examples/kubernetes/nfs
```

Create namespace for the rook-nfs operator
`k create ns rook-nfs-system`

Create namespace for the rook-nfs server
`k create ns rook-nfs`

Create the Operator by deploying the below resources

`kubectl create -f nfs/crds.yaml`
`kubectl create -f nfs/operator.yaml`

Validate the operator is up and running

`kubectl -n rook-nfs-system get pod`

# Deploy NFS Admission Webhook

Admission webhooks are HTTP callbacks that receive admission requests to the API server.
Two types of admission webhooks is validating admission webhook and mutating admission webhook.
NFS Operator support validating admission webhook which validate the NFSServer object sent to the API server before stored in the etcd (persisted).

Prerequisite for installing the admission Webhook is to have Cert-Manager installed. Once this is meet deploy the NFS webhook.

`kubectl create -f nfs/webhook.yaml`

Verify the webhook is up and running

`kubectl -n rook-nfs-system get pod`

# Create Pod Security Policies

`kubectl create -f nfs/psp.yaml`

# Create the required Service Account and roles

NFS Server we need to create ServiceAccount and RBAC rules

`kubectl apply -f nfs/rbac.yaml -n rook-nfs`

# Create and Initialize NFS Server using the Default StorageClass example

Update the definition file below with the relevant default storage class

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-default-claim
  namespace: rook-nfs
spec:
  storageClassName: linode-block-storage-retain
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

Update the definition file and change the export name to share instead of share1

```
apiVersion: nfs.rook.io/v1alpha1
kind: NFSServer
metadata:
  name: rook-nfs
  namespace: rook-nfs
spec:
  replicas: 1
  exports:
    - name: share
      server:
        accessMode: ReadWrite
        squash: 'none'
      # A Persistent Volume Claim must be created before creating NFS CRD instance.
      persistentVolumeClaim:
        claimName: nfs-default-claim
  # A key/value list of annotations
  annotations:
    rook: nfs

```

# Create the NFS Server CRD

`kubectl create -f nfs/nfs.yaml`

# Consuming the storage

Once the NFS Operator and the NFS Server is deployed you can consume storage in the cluster.
Update the storageClass parameter exportName to share instead of share1

```
parameters:
  exportName: share
```

Create a storageClass using the example defintion file to dynamically provision volumes.
This creates a SC with reclaimPolicy: Delete -- Best suited for apps with no data retention

`kubectl create -f nfs/sc.yaml`

This creates a SC with reclaimPolicy: Retain -- Best suited for statefullset

`kubectl create -f nfs/scr.yaml`

# Validate both StorageClasse are created

Remember sc are cluster scoped

`K get sc`

# Test the install is successful by deploying the sample app in the test folder

`k apply -Rf test/`

**\_\__ OPTION-2 _\_\_**

# Update your helm values and deploy using kubectl apply

Reference the helm values `rook-ceph-test-helm-values` for all the required changes.

# Render chart locally and validate add --namespace flag to generate the chart for particular namespace

`helm template rook-ceph --namespace rook-ceph --values rook-ceph-test-helm-values.yaml --output-dir ./manifests/test/ rook-ceph`

# Apply all the rendered yaml files using kubectl apply

`k apply -Rf manifests/test/rook-ceph -n rook-ceph`

# Un-install - Delete all the rendered yaml files

`k delete -Rf manifests/test/rook-ceph -n rook-ceph`

# Use the Helm upgrade --install.

https://rook.io/docs/rook/v1.8/helm-operator.html#configuration Ceph-Operator

https://rook.io/docs/rook/v1.8/helm-ceph-cluster.html#release Ceph-Cluster

`helm upgrade --install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph `

# Un-install using helm uninstal - Delete all the objects created

`helm uninstall rook-ceph -n rook-ceph`

_POST INSTALL CONFIGURATION_

**TIPS**

I was not able to delete the ns rook-ceph uninstalling rook operator
It was because there were zombies resources still in the namespace
To resolve I ran the below code.

```
    NAMESPACE=your-rogue-namespace
    kubectl proxy &
    kubectl get namespace $NAMESPACE -o json |jq '.spec = {"finalizers":[]}' >temp.json
    curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
```

Also had to recreate the ns to update the below resources to remove `finalizer` Metadata

`kubectl patch configmap -n rook-ceph rook-ceph-mon-endpoints -p '{"metadata":{"finalizers":null}}'`

`kubectl patch secret -n rook-ceph rook-ceph-mon -p '{"metadata":{"finalizers":null}}'`
