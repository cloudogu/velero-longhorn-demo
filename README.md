# Cluster Backups mit Velero & Longhorn

## Einleitung
Nach der Migration des [Cloudogu EcoSystem](https://cloudogu.com/de/ecosystem/) auf [Kubernetes](https://cloudogu.com/de/glossar/kubernetes/) sollen unter anderem auch Backups wieder unterstützt werden.

Im Open Source Bereich ist Velero als Tool für Cluster Backups praktisch alternativlos.
Es kann alle K8s-Ressourcen, aber auch Volume Daten sichern.
Backups lassen sich mit Schedules automatisch ausführen.
Erweiterbar ist es durch [Plugins](https://velero.io/docs/main/custom-plugins): So lässt sich zum Beispiel auch ein [S3 Bucket anbinden](https://github.com/vmware-tanzu/velero-plugin-for-aws).
Wer einen anderen Storage Provider als den Cluster-Internen benutzt, also zum Beispiel Longhorn,
der kann diesen mit einem [Plugin für das Container-Storage-Interface](https://github.com/vmware-tanzu/velero-plugin-for-csi) (im Folgenden CSI) anbinden.

Im Folgenden wollen wir genau dies tun:
Wir installieren Longhorn und Velero.
Beides konfigurieren wir so, dass wir Backups von Teilen unseres Clusters inklusive Volume-Daten auf ein S3 (in unserem Fall MinIO) schreiben können.

## Funktionsweise

![Schaubild, das die Funktionsweise von Velero mit Longhorn darstellt](figures/Velero%20Longhorn%20Backups.drawio.svg "Schaubild: Funktionsweise von Velero")

Bei Velero handelt es sich um einen [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).
Ein Operator hört auf das Erstellen, Ändern und Löschen [bestimmter Kubernetes Ressourcen](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), und verarbeitet diese.
Velero hört beispielsweise auf Ressourcen vom Typ [`velero.io/v1/Backup`](https://velero.io/docs/main/api-types/backup/) und [`velero.io/v1/Restore`](https://velero.io/docs/main/api-types/restore/).

Damit diese Ressourcen nicht manuell angelegt werden müssen, kommt Velero mit einem kleinen CLI-Tool, welches diese Aufgabe übernimmt.

### Backup
Wird also der Befehl `velero backup create <backup-name>` aufgerufen, wird eine neue [Backup-Ressource](https://velero.io/docs/main/api-types/backup/) mit dem angegebenen Namen angelegt.
Der Velero Server erkennt die angelegte Ressource und startet den Backup-Prozess.
Alle im Backup angegebenen Ressourcen werden gesammelt und gespeichert.
Soll das Backup außerhalb des Clusters gespeichert werden, muss ein Object Store Plugin eingebunden werden.
In unserem Fall ist es das [Velero Plugin for AWS](https://github.com/vmware-tanzu/velero-plugin-for-aws), welches die Daten auf ein S3 Bucket schreibt.

#### Doch was passiert mit den Volume Daten?
Verwendet man ganz normale Kubernetes Volumes, werden diese zusammen mit den anderen Ressourcen von Velero in das MinIO Bucket geschrieben.  
Da wir jedoch Longhorn verwenden, ist dies nicht möglich.
Allerdings unterstützt Longhorn das Container Storage Interface, welches es uns ermöglicht, Longhorn mitzuteilen, dass es ein Backup erstellen soll.  
Hier kommt das [Velero Plugin for CSI](https://github.com/vmware-tanzu/velero-plugin-for-csi) ins Spiel:
Es erstellt einen [`VolumeSnapshot`](https://kubernetes.io/docs/concepts/storage/volume-snapshots/), was Longhorn dazu veranlasst, ein Backup zu erstellen.
Wenn Longhorn nun richtig konfiguriert ist, wird dieses Backup in ein S3 Bucket geschrieben.

### Restore
Wird der Befehl `velero restore create --from-backup <backup-name>` aufgerufen, wird eine neue [Restore-Ressource](https://velero.io/docs/main/api-types/restore/) vom angegebenen Backup angelegt.
Die Ressource wird von Velero erkannt und das entsprechende Backup aus dem S3 Bucket gezogen.

Um das Backup der Volumes einzuspielen, muss die `dataSource` des `PersistentVolumeClaims` auf den entsprechenden `VolumeSnapshot` zeigen, anstatt auf ein `PersistentVolume`.
Auch dies erfolgt automatisch durch das Velero-CSI-Plugin.

## Anleitung

### Voraussetzungen
- Git
- Docker
- [kubectl](https://kubernetes.io/de/docs/tasks/tools/install-kubectl/)
- [kind (Kubernetes in Docker)](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Helm](https://helm.sh/docs/intro/install/)

Nützlich ist zum Beobachten unseres Clusters ist außerdem [K9s](https://k9scli.io/).

### Repository klonen
```shell
git clone git@github.com:cloudogu/velero-longhorn-demo.git
```

Alle weiteren Befehle werden im Hauptverzeichnis des Repositories ausgeführt.

### Cluster aufsetzen

Für unser Beispiel verwenden wir ein kind Cluster.
Dieses kann wie folgt aufgesetzt werden:
```shell
kind create cluster --config src/cluster-config.yaml
```

### MinIO einrichten
Ein lokales MinIO lässt sich mit folgendem Befehl starten:
```shell
docker run -d --name minio \
    -p 9000:9000 -p 9090:9090 \
    -e "MINIO_ROOT_USER=MINIOADMIN" \
    -e "MINIO_ROOT_PASSWORD=MINIOADMINPW" \
    quay.io/minio/minio \
    server /data --console-address ":9090"
```

Jetzt können wir die MinIO Administration unter http://localhost:9000 öffnen.
Anmelden kann man sich mit den in den obigen Umgebungsvariablen angegebenen Credentials.

Wir legen zwei Buckets an: `longhorn` und `velero`.  
Außerdem erstellen wir einen Access Key.
In unserem Beispiel sieht dieser so aus:
- Key ID: `test-key`
- Secret Key: `secret-test-key`.

### Longhorn

#### Longhorn installieren
Longhorn installieren wir nach der [offiziellen Installationsanweisung](https://longhorn.io/docs/1.4.0/deploy/install/).  
In unserem Fall sieht das so aus:
```shell
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn \
    longhorn/longhorn \
    --namespace longhorn-system \
    --create-namespace \
    --values src/longhorn-values.yaml \
    --version 1.4.0
```

Nun sollten nach und nach die Pods im `longhorn-system` Namespace starten.  
Falls es zu Problemen kommt, sollte überprüft werden,
ob [open-iscsi](https://longhorn.io/docs/1.4.0/deploy/install/#installing-open-iscsi) und
der [NFSv4 client](https://longhorn.io/docs/1.4.0/deploy/install/#installing-nfsv4-client) installiert und aktiviert sind.

#### Longhorn konfigurieren
Bei der Installation haben wir Longhorn schon so konfiguriert, dass es die Zugangsdaten für den Ablageort des Backups aus dem `minio-secret` liest.
Dieses Secret müssen wir nun noch anwenden:
```shell
kubectl apply -f src/longhorn-minio-secret.yaml
```

> **Notiz:** Im Secret benutzen wir die IP `172.17.0.1`, um den Host zu erreichen.
> Dies funktioniert allerdings nur unter Linux und
> wenn sich das KIND cluster im `docker0` Netzwerk befindet (default).
> Unter Windows und Mac kann `host.docker.internal` verwendet werden.

### Snapshot Controller und CSI Snapshot CRDs
Um CSI Snapshots erstellen zu können, werden ein Snapshot-Controller und die CSI Snapshot CRDs benötigt.
Da KIND diese standardmäßig nicht installiert hat, müssen wir diese manuell installieren:
```shell
kubectl -n kube-system create -k "git@github.com:kubernetes-csi/external-snapshotter/client/config/crd?ref=release-6.2"
kubectl -n kube-system create -k "git@github.com:kubernetes-csi/external-snapshotter/deploy/kubernetes/snapshot-controller?ref=release-6.2"
```

Damit Longhorn für die Snapshots verwendet wird, müssen wir eine `VolumeSnapshotClass` anlegen.
Diese sieht folgendermaßen aus:
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

Damit auch Velero bei den `VolumeSnapshots` Longhorn verwendet, müssen wir das Label `velero.io/csi-volumesnapshot-class: "true"` anhängen.

Unter `parameters` geben wir `type: bak` an, was Longhorn sagt, dass wir ein Longhorn Backup machen möchten.
Eine Alternative wäre `type: snap` für Longhorn Snapshots (inkrementelle Backups, nicht zu verwechseln mit den `VolumeSnapshots`).
Diese haben jedoch bisher noch keine CSI-Unterstützung, können also hier nicht verwendet werden.

Die `VolumeSnapshotClass` wenden wir nun also gegen das Cluster an:
```shell
kubectl apply -f src/default-volumesnapshotclass.yaml
```

### Velero

#### Velero CLI

Das Velero CLI installieren wir wie in der [offiziellen Dokumentation](https://velero.io/docs/v1.10/basic-install/#option-2-github-release) angegeben über das [GitHub Release](https://github.com/vmware-tanzu/velero/releases/tag/v1.10.0).

#### Velero Server

Velero kann folgendermaßen mit Helm installiert werden:
```shell
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
helm install velero \
    --namespace=velero \
    --create-namespace \
    --set-file credentials.secretContents.cloud=src/credentials-velero \
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

Wir konfigurieren hier auch direkt schon die Plugins für S3 und CSI.
Die MinIO Credentials werden dabei direkt aus der Datei [`src/credentials-velero`](src/credentials-velero) ausgelesen.
`snapshotEnabled=true` und `configuration.features=EnableCSI` sind beide nötig um die CSI `VolumeSnapshot` Unterstützung zu aktivieren.
Wie wir sehen können, sind Velero Plugins als `InitContainer` umgesetzt.

### Testen von Backup und Restore

Um Backup und Restore zu testen, installieren wir eine kleine Testanwendung.
Diese besteht aus einem simplen Pod, der ein Longhorn Volume aus einem PVC mountet.
```shell
kubectl apply -f src/example-app.yaml
```

Nun schreiben wir `Hello from Velero!` in eine Datei im Volume:
```shell
kubectl -n csi-app exec -ti csi-nginx -- bash -c 'echo -n "Hello from Velero!" >> /mnt/longhorndisk/hello'
```

Dann machen wir ein Backup:
```shell
velero backup create csi-b1 --include-namespaces csi-app --wait
```

Desaster! Aus Versehen wurde der `csi-app` Namespace gelöscht!
```shell
kubectl delete ns csi-app
```

Puh! Gut, dass wir ein Backup haben! Dieses wollen wir nun wieder einspielen:
```shell
velero restore create --from-backup csi-b1 --wait
```

Wenn alles funktioniert hat, sollte folgender Befehl uns nun `Hello from Velero!` aus der zuvor erstellten Datei ausgeben:
```shell
kubectl -n csi-app exec -ti csi-nginx -- bash -c 'cat /mnt/longhorndisk/hello'
```

## Findings

### Longhorn funktioniert nicht mit K3d
Ich habe zunächst versucht, K3d (K3s in Docker) statt KIND zu verwenden.
Jedoch hat dort die Installation von Longhorn nicht funktioniert,
wahrscheinlich wegen fehlendem [open-iscsi](https://longhorn.io/docs/1.4.0/deploy/install/#installing-open-iscsi)
und [NFSv4 client](https://longhorn.io/docs/1.4.0/deploy/install/#installing-nfsv4-client).