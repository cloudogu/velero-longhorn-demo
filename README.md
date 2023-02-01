# Cluster Backups mit Velero & Longhorn

## Einleitung
Aktuell arbeiten wir daran, das [Cloudogu EcoSystem](https://cloudogu.com/de/ecosystem/) auf Kubernetes zu migrieren.
Da auch das bisherigen Cloudogu EcoSystem Backups und Restores unterstützt, soll dies in Zukunft auch möglich sein.

Für unsere Zwecke ist [Velero](https://velero.io) praktisch alternativlos.
Es kann alle K8s-Ressourcen, aber auch Volume-Daten sichern.
Backups lassen sich mit Schedules automatisch ausführen.
Erweiterbar ist es durch [Plugins](https://velero.io/docs/main/custom-plugins): So lässt sich zum Beispiel auch ein [S3 Bucket anbinden](https://github.com/vmware-tanzu/velero-plugin-for-aws).
Wer einen anderen Storage-Provider als den Cluster-Internen benutzt, also zum Beispiel Longhorn,
der kann diesen mit einem [Plugin für das Container-Storage-Interface (CSI)](https://github.com/vmware-tanzu/velero-plugin-for-csi) anbinden.

Im Folgenden wollen wir genau dies tun:
Wir installieren Longhorn und Velero.
Beides konfigurieren wir so, dass wir Backups von Teilen unseres Clusters inklusive Volume-Daten auf ein S3 (in unserem Fall MinIO) schreiben können.

## Funktionsweise

![Schaubild, das die Funktionsweise von Velero mit Longhorn darstellt](figures/Velero%20Longhorn%20Backups.drawio.svg "Schaubild: Funktionsweise von Velero")

Bei Velero handelt es sich um einen [Kubernetes-Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).
Ein Operator hört auf das Erstellen, Ändern und Löschen [bestimmter Kubernetes-Ressourcen](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), und verarbeitet diese.
Velero hört beispielsweise auf Ressourcen vom Typ [`velero.io/v1/Backup`](https://velero.io/docs/main/api-types/backup/) und [`velero.io/v1/Restore`](https://velero.io/docs/main/api-types/restore/).

Damit diese Ressourcen nicht manuell angelegt werden müssen, kommt Velero mit einem kleinen CLI-Tool, welches diese Aufgabe übernimmt.

### Backup
Der Befehl `velero backup create <backup-name>` legt eine neue [Backup-Ressource](https://velero.io/docs/main/api-types/backup/) mit dem angegebenen Namen an.
Der Velero-Server erkennt die angelegte Ressource und startet den Backup-Prozess,
der alle im Backup angegebenen Ressourcen sammelt und speichert.
Zur Speicherung des Backups außerhalb des Clusters können wir ein Object-Store-Plugin einbinden.
In unserem Fall ist es das [Velero-Plugin-for-AWS](https://github.com/vmware-tanzu/velero-plugin-for-aws), welches die Daten auf ein S3-Bucket schreibt.

#### Doch was passiert mit den Volume-Daten?
Verwendet man ganz normale Kubernetes-Volumes, schreibt Velero diese zusammen mit den anderen Ressourcen in das MinIO-Bucket.  
Da wir jedoch Longhorn verwenden, ist dies nicht möglich.
Allerdings unterstützt Longhorn das Container-Storage-Interface, über welches Velero Longhorn mitteilen kann, dass es ein Backup erstellen soll.  
Hier kommt das [Velero-Plugin-for-CSI](https://github.com/vmware-tanzu/velero-plugin-for-csi) ins Spiel:
Es erstellt einen [`VolumeSnapshot`](https://kubernetes.io/docs/concepts/storage/volume-snapshots/), was Longhorn dazu veranlasst, ein Backup zu erstellen.
Wenn Longhorn nun richtig konfiguriert ist, schreibt es dieses Backup in ein S3-Bucket.

### Restore
Der Befehl `velero restore create --from-backup <backup-name>` legt eine neue [Restore-Ressource](https://velero.io/docs/main/api-types/restore/) vom angegebenen Backup an.
Velero erkennt die Ressource und wendet das entsprechende Backup aus dem S3-Bucket auf das Cluster an.

Um das Backup der Volumes einzuspielen, muss die `dataSource` des `PersistentVolumeClaims` auf den entsprechenden `VolumeSnapshot` zeigen, anstatt auf ein `PersistentVolume`.
Auch das übernimmt das Velero-CSI-Plugin.

## Anleitung

### Voraussetzungen
- Git
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Vagrant](https://developer.hashicorp.com/vagrant/downloads) >= 2.3.4

### Repository klonen
```shell
git clone git@github.com:cloudogu/velero-longhorn-demo.git && cd velero-longhorn-demo
```

### Cluster aufsetzen

Für unser Beispiel verwenden wir ein K3s-Cluster, dieses wird folgendermaßen gestartet:
```shell
vagrant up
```

Nun verbinden wir uns per SSH mit dem Cluster:
```shell
vagrant ssh
```

Natürlich ist es auch möglich, ein eigenes Cluster zu verwenden.

> **Notiz:** Longhorn funktioniert nicht mit K3d oder KIND  
> Ich habe sowohl K3d (K3s in Docker) als auch KIND (Kubernetes in Docker) ausprobiert.
> Jedoch [funktioniert `iscsi` nicht in Containern](https://github.com/longhorn/longhorn/discussions/2702),
> weswegen Longhorn dort nicht funktioniert.

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

Jetzt können wir die MinIO-Administration unter http://localhost:9000 öffnen.
Anmelden können wir uns mit den in den obigen Umgebungsvariablen angegebenen Credentials.

Wir legen zwei Buckets an: `longhorn` und `velero`.  
Außerdem erstellen wir einen Access-Key:
- Key ID: `test-key`
- Secret Key: `test-secret-key`.

### Longhorn

#### Longhorn installieren
Longhorn installieren wir nach der [offiziellen Installationsanweisung](https://longhorn.io/docs/1.4.0/deploy/install/):
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

In der [`longhorn-values.yaml`](src/longhorn-values.yaml) konfigurieren wir auch das Backup-Target von Longhorn, sodass Longhorn die Volume Backups auf das S3 schreibt.
In der Praxis können wir dies aber auch im Nachhinein z.B. über die Longhorn-UI konfigurieren.

Nun sollten nach und nach die Pods im `longhorn-system` Namespace starten.
Um das Cluster zu beobachten, ist das Tool `k9s` sehr praktisch, welches auf der VM schon vorinstalliert ist.

#### Longhorn konfigurieren
Bei der Installation haben wir Longhorn schon so konfiguriert, dass es die Zugangsdaten für den Ablageort des Backups aus dem `minio-secret` liest.
Dieses Secret müssen wir nun noch anwenden:
```shell
kubectl apply -f /vagrant/src/longhorn-minio-secret.yaml
```

> **Notiz:** Im Secret benutzen wir die IP `172.17.0.1`, um den Host zu erreichen.  
> Ob dies möglich ist, hängt von der Container-Runtime und Netzwerk-Konfiguration ab.

### Snapshot-Controller und CSI-Snapshot-CRDs
Um CSI-Snapshots zu erstellen, benötigen wir einen Snapshot-Controller und die CSI-Snapshot-CRDs.
Da diese auf K3s standardmäßig nicht installiert sind, müssen wir sie manuell installieren:
```shell
kubectl -n kube-system create -k "github.com/kubernetes-csi/external-snapshotter/client/config/crd?ref=release-5.0"
kubectl -n kube-system create -k "github.com/kubernetes-csi/external-snapshotter/deploy/kubernetes/snapshot-controller?ref=release-5.0"
```

> **Notiz:** [Longhorn unterstützt bisher keine Versionen des Snapshot-Controllers neuer als 5.0.](https://github.com/longhorn/longhorn-manager/pull/1518#discussion_r991818681)


Die `VolumeSnapshotClass` wenden wir nun also gegen das Cluster an:
```shell
kubectl apply -f /vagrant/src/default-volumesnapshotclass.yaml
```

Damit der Snapshot-Controller Longhorn für die Snapshots verwendet, müssen wir eine `VolumeSnapshotClass` anlegen.
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

Damit Velero die `VolumeSnapshots` mit unserer `VolumeSnapshotClass` erstellt, benötigen wir das Label `velero.io/csi-volumesnapshot-class: "true"`.

Unter `parameters` geben wir `type: bak` an, was Longhorn sagt, dass wir ein Longhorn-Backup machen möchten.
Eine Alternative wäre `type: snap` für Longhorn-Snapshots (inkrementelle Backups, nicht zu verwechseln mit den `VolumeSnapshots`).
Diese haben jedoch bisher noch keine CSI-Unterstützung, können also hier nicht verwendet werden.

### Velero

#### Velero-CLI

Das Velero-CLI installieren wir wie in der [offiziellen Dokumentation](https://velero.io/docs/v1.10/basic-install/#option-2-github-release) angegeben über das [GitHub Release](https://github.com/vmware-tanzu/velero/releases/tag/v1.10.0).:
```shell
VELERO_VERSION=v1.10.0; \
    wget -c https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz -O - \
    | tar -xz -C /tmp/ \
    && sudo mv /tmp/velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin
```

#### Velero-Server

Velero installieren wir mit Helm:
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

Wir konfigurieren hier auch direkt schon die Plugins für S3 und CSI.
Die MinIO-Credentials werden dabei direkt aus der Datei [`src/credentials-velero`](src/credentials-velero) ausgelesen.
`snapshotEnabled=true` und `configuration.features=EnableCSI` sind beide nötig um die CSI `VolumeSnapshot` Unterstützung zu aktivieren.

### Testen von Backup und Restore

Um Backup und Restore zu testen, installieren wir eine kleine Testanwendung.
Diese besteht aus einem simplen Pod, der ein Longhorn-Volume aus einem PVC mountet.
```shell
kubectl apply -f /vagrant/src/example-app.yaml
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

## Fazit

Velero ist ein gutes und ausgereiftes Tool um Cluster Backups zu erstellen.
Durch das Container-Storage-Interface ist es unabhängig vom Storage-Provider, solange dieser CSI unterstützt.

Ein Feature, welches Velero noch nicht unterstützt, sind Backups mit Ende-zu-Ende-Verschlüsselung.
Da Velero Open-Source ist, ist eine Erweiterung in diese Richtung jedoch nicht ausgeschlossen.
Falls dies passiert, könnt ihr euch auf einen weiteren Blog-Post freuen!

Rundum macht die asynchrone Funktionsweise und die Operator-Architektur Velero wie geschaffen dafür, in andere Produkte integriert zu werden.
Deswegen glauben wir von Cloudogu, dass Velero die beste Backup-Lösung für das [Cloudogu K8s-EcoSystem](https://github.com/cloudogu/k8s-ecosystem/) ist.
