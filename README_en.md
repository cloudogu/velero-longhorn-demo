# Cluster Backups with Velero & Longhorn

## Why Verlero?
We are currently working on migrating the [Cloudogu EcoSystem](https://cloudogu.com/de/ecosystem/) to Kubernetes.
Since the current Cloudogu EcoSystem supports backups and restores, this should also be possible after the migration to Kubernetes.

For our purposes, [Velero](https://velero.io) is practically without alternative.
It can back up all K8s resources, but also volume data.
Backups can be executed automatically with schedules.
It can be extended with [plugins](https://velero.io/docs/main/custom-plugins): For example, a [S3 bucket can be connected](https://github.com/vmware-tanzu/velero-plugin-for-aws).
If, like us, you do not use the cluster-internal storage provider, but for example Longhorn, you can easily [connect it with a plug-in for the container storage interface (CSI)](https://github.com/vmware-tanzu/velero-plugin-for-csi).

In the following we will do exactly this:
We install Longhorn and Velero.
We configure both so that we can write backups of parts of our cluster including data from volumes to an S3 (in our case MinIO).

## How Backup & Restore works

![Diagram showing how Velero and Longhorn work together](figures/Velero%20Longhorn%20Backups.drawio.svg "Diagram: How Velero works")

Velero is a [Kubernetes operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).
An operator listens for and processes the creation, modification, and deletion of [specific Kubernetes resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).
For example, Velero listens for resources of type [`velero.io/v1/Backup`](https://velero.io/docs/main/api-types/backup/) and [`velero.io/v1/Restore`](https://velero.io/docs/main/api-types/restore/).

To avoid having to create these resources manually, Velero comes with a small CLI tool that handles this task.

### Backup
The `velero backup create <backup-name>` command creates a new [backup resource](https://velero.io/docs/main/api-types/backup/) with the specified name.
The Velero server recognizes the created resource and starts the backup process,
which collects and stores all the resources specified in the backup.
To store the backup outside the cluster, we can include an object store plugin.
In our case, it is the [Velero-plugin-for-AWS](https://github.com/vmware-tanzu/velero-plugin-for-aws), which writes the data to a S3 bucket.

#### But what happens to the volume data?
If you use normal Kubernetes volumes, Velero writes them to the MinIO bucket along with the other resources.
Since we are using Longhorn, this is not possible.
However, Longhorn supports the container storage interface, through which Velero can tell Longhorn to create a backup.
This is where the [Velero-plugin-for-CSI](https://github.com/vmware-tanzu/velero-plugin-for-csi) comes in:
it creates a [`VolumeSnapshot`](https://kubernetes.io/docs/concepts/storage/volume-snapshots/), which tells Longhorn to create a backup.
Now, if Longhorn is configured correctly, it writes this backup to a S3 bucket.

### Restore
The `velero restore create --from-backup <backup-name>` command creates a new [restore resource](https://velero.io/docs/main/api-types/restore/) from the specified backup.
Velero recognizes the resource and applies the corresponding backup from the S3 bucket to the cluster.

To ingest the backup of the volumes, the `dataSource` of the `PersistentVolumeClaim` must point to the corresponding `VolumeSnapshot` instead of a `PersistentVolume`.
This is also handled by the Velero CSI plugin.

## Instructions for Backup & Restore with Velero and Longhorn

All files and examples are available [on GitHub](https://github.com/cloudogu/velero-longhorn-demo).

### Requirements
- Git
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Vagrant](https://developer.hashicorp.com/vagrant/downloads) >= 2.3.4

### Clone repository
```shell
git clone git@github.com:cloudogu/velero-longhorn-demo.git && cd velero-longhorn-demo
```

### Cluster aufsetzen

For our example we will use a K3s cluster, which is started as follows:
```shell
vagrant up
```

Now we connect to the cluster via SSH:
```shell
vagrant ssh
```

Of course, it is also possible to use your own cluster.

> **Note:** Longhorn does not work with K3d or KIND  
> I have tried both K3d (K3s in Docker) and KIND (Kubernetes in Docker).
> However, [`iscsi` does not work in containers](https://github.com/longhorn/longhorn/discussions/2702),
> which is why Longhorn does not work there.

### Setting up MinIO
A local MinIO can be started with the following command:
```shell
docker run -d --name minio \
    -p 9000:9000 -p 9090:9090 \
    -e "MINIO_ROOT_USER=MINIOADMIN" \
    -e "MINIO_ROOT_PASSWORD=MINIOADMINPW" \
    quay.io/minio/minio \
    server /data --console-address ":9090"
```

Now we can open the MinIO administration at http://localhost:9000.
We can log in with the credentials specified in the environment variables above.

We create two buckets: `longhorn` and `velero`.  
We also create an access key:
- Key ID: `test-key`
- Secret Key: `test-secret-key`.

### Longhorn

#### Longhorn installieren
We install Longhorn according to the [official installation instructions](https://longhorn.io/docs/1.4.0/deploy/install/):
```shell
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn \
    longhorn/longhorn \
    --namespace longhorn-system \
    --create-namespace \
    --values /vagrant/src/longhorn-values.yaml \
    --version 1.4.0
```

In [`longhorn-values.yaml`](src/longhorn-values.yaml), we also configure Longhorn's backup target so that Longhorn writes the volume backups to the S3.
In practice, however, we can also configure this afterwards, for example via the Longhorn UI.

Now, one by one, the pods should start in the `longhorn-system` namespace.
To monitor the cluster, the tool `k9s`  is very practical, which is already pre-installed on the VM.

#### Configure Longhorn
During installation, we already configured Longhorn to read the credentials for the backup location from the `minio-secret`.
Now we have to apply this secret:
```shell
kubectl apply -f /vagrant/src/longhorn-minio-secret.yaml
```

> **Note:** In the secret we use the IP ‘172.17.0.1’ to reach the host.  
> Whether this is possible depends on the container runtime and network configuration.
> In a real-life scenario, the backup would be stored off-site anyway.

### Snapshot Controller and CSI Snapshot CRDs
To create CSI snapshots, we need a snapshot controller and the CSI snapshot CRDs.
Since these are not installed by default on K3s, we need to install them manually:
```shell
kubectl -n kube-system create -k "github.com/kubernetes-csi/external-snapshotter/client/config/crd?ref=release-5.0"
kubectl -n kube-system create -k "github.com/kubernetes-csi/external-snapshotter/deploy/kubernetes/snapshot-controller?ref=release-5.0"
```

> **Note:** [Longhorn does not yet support versions of the snapshot controller newer than 5.0.](https://github.com/longhorn/longhorn-manager/pull/1518#discussion_r991818681)


In order for the snapshot controller to use Longhorn for the snapshots, we need to create a `VolumeSnapshotClass`:
```shell
kubectl apply -f /vagrant/src/default-volumesnapshotclass.yaml
```

This looks like this:
```yaml
kind: VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
metadata:
  name: longhorn-snapshot-vsc
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: driver.longhorn.io
deletionPolicy: Delete
parameters:
  type: bak
```

To make Velero create the `VolumeSnapshots` with our `VolumeSnapshotClass` we need the label `velero.io/csi-volumesnapshot-class: "true"`.

Under `parameters`, we specify `type: bak`, which tells Longhorn that we want to make a Longhorn backup.
An alternative would be `type: snap` for Longhorn snapshots (incremental backups, not to be confused with `VolumeSnapshots`).
However, these do not yet have CSI support, so cannot be used here.

### Velero

#### Velero CLI

We install the Velero CLI as stated in the [official documentation](https://velero.io/docs/v1.10/basic-install/#option-2-github-release) via the [GitHub release](https://github.com/vmware-tanzu/velero/releases/tag/v1.10.0):
```shell
VELERO_VERSION=v1.10.0; \
    wget -c https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz -O - \
    | tar -xz -C /tmp/ \
    && sudo mv /tmp/velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin
```

#### Velero server

We install Velero with Helm:
```shell
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
helm install velero \
    --namespace=velero \
    --create-namespace \
    --set-file credentials.secretContents.cloud=/vagrant/src/credentials-velero \
    --set configuration.provider=aws \
    --set configuration.backupStorageLocation.name=default \
    --set configuration.backupStorageLocation.bucket=velero \
    --set configuration.backupStorageLocation.config.region=minio-default \
    --set configuration.backupStorageLocation.config.s3ForcePathStyle=true \
    --set configuration.backupStorageLocation.config.s3Url=http://172.17.0.1:9000 \
    --set configuration.backupStorageLocation.config.publicUrl=http://localhost:9000 \
    --set snapshotsEnabled=true \
    --set configuration.volumeSnapshotLocation.name=default \
    --set configuration.volumeSnapshotLocation.config.region=minio-default \
    --set "initContainers[0].name=velero-plugin-for-aws" \
    --set "initContainers[0].image=velero/velero-plugin-for-aws:v1.6.0" \
    --set "initContainers[0].volumeMounts[0].mountPath=/target" \
    --set "initContainers[0].volumeMounts[0].name=plugins" \
    --set configuration.features=EnableCSI \
    --set "initContainers[1].name=velero-plugin-for-csi" \
    --set "initContainers[1].image=velero/velero-plugin-for-csi:v0.4.0" \
    --set "initContainers[1].volumeMounts[0].mountPath=/target" \
    --set "initContainers[1].volumeMounts[0].name=plugins" \
    vmware-tanzu/velero
```

We also configure the plugins for S3 and CSI here.
The MinIO credentials are read directly from the file [`src/credentials-velero`](src/credentials-velero).
`snapshotEnabled=true` and `configuration.features=EnableCSI` are both necessary to enable CSI `VolumeSnapshot` support.

### Testing backup and restore

To test backup and restore, we install a small test application.
This consists of a simple pod that mounts a Longhorn volume from a PVC.
```shell
kubectl apply -f /vagrant/src/example-app.yaml
```

Now we write `Hello from Velero!` to a file in the volume:
```shell
kubectl -n csi-app exec -ti csi-nginx -- bash -c 'echo -n "Hello from Velero!" >> /mnt/longhorndisk/hello'
```

Then we make a backup:
```shell
velero backup create csi-b1 --include-namespaces csi-app --wait
```

Disaster! By mistake the `csi-app` namespace was deleted!
```shell
kubectl delete ns csi-app
```

Phew! Good that we have a backup! Now we want to restore it:
```shell
velero restore create --from-backup csi-b1 --wait
```

If everything worked, the following command should now output `Hello from Velero!`  from the previously created file:
```shell
kubectl -n csi-app exec -ti csi-nginx -- bash -c 'cat /mnt/longhorndisk/hello'
```

## Conclusion

Velero is a good and mature tool to create cluster backups.
The container storage interface makes it independent of the storage provider as long as it supports CSI.

One feature that Velero does not yet support is backups with end-to-end encryption.
However, since Velero is open-source, an extension in this regard is not out of the question.
If this happens, you can look forward to another blog post!

Thanks to asynchronous functionality as well as the operator architecture, Velero integrates perfectly with other products.
That's why we at Cloudogu believe that Velero is the best backup solution for the [Cloudogu K8s EcoSystem](https://github.com/cloudogu/k8s-ecosystem/).
