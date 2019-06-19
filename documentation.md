# Disaster Management with Heptio-Velero 

### A Quick info on Velero. 

[Velero](https://velero.io/docs/v0.11.0/index.html) (formerly Heptio Ark) gives you tools to back up and restore your Kubernetes cluster resources, It also takes snapshots of your cluster’s Persistent Volumes using your cloud provider’s block storage snapshot features, and can then restore your cluster’s objects and Persistent Volumes to a previous state using [Restic](https://velero.io/docs/v0.11.0/restic/). One can use Velero to perform full backups, only backups of some namespaces or resource types or you can schedule backups to execute them periodically.

For detail explanation of how Velero works follow the [link](https://velero.io/docs/v0.11.0/about/)

#### Why we have to take the Backups of Kubernetes?

* To Recover from the Disasters such as
   * When someone deleted the namespace accidentally.
   * When the Network is down.
   * When cluster goes into an unrecoverable state.
   * When a bug cleaned out your PV and all your data is lost.
* To Migrate the Kubernetes cluster from one environment to another.
* To Replicate the environment before upgrade.

## Prerequisites

* Access to a Kubernetes cluster, version 1.7 or later. Version 1.7.5 or later is required to run Velero.
* kubectl installed

#### Setup Velero on your cluster:

This [link](https://velero.io/docs/v0.11.0/ibm-config/) explains how to setup Velero on IBM Cloud with IBM Cloud Object Storage as Velero's storage destination. 

* Download an official [release](https://github.com/heptio/velero/releases) of Velero
  * To Install Velero run the 'velero-install.sh' which extracts the files needed for the IBM cloud. 
  * Check the latest releases of the Heptio-Velero here [link](https://github.com/heptio/velero/releases)
* Create your [COS instance](https://cloud.ibm.com/docs/services/cloud-object-storage/basics?topic=cloud-object-storage-provision#creating-a-new-resource-instance).
* Create an [S3 bucket](https://cloud.ibm.com/docs/services/cloud-object-storage?topic=cloud-object-storage-getting-started#create-buckets)
* Define a [service](https://cloud.ibm.com/docs/services/cloud-object-storage/iam?topic=cloud-object-storage-service-credentials#service-credentials) that can store data in the bucket. Follow each and every step listed in the official document [here](https://velero.io/docs/v0.11.0/ibm-config/)  
* Configure and start the Velero [server](https://velero.io/docs/v0.11.0/ibm-config/)
  * It is important to know what's going on behind the scenes before apply the pre-req's in the above link. 
    * Backup: CRD that stores information such as creation date, which namespaces should be included, which PVCs are attached, and so on.
	* BackupLocation: CRD that stores information such as which region and object storage should be used to store backups.
	* SnapshotLocation: CRD that stores information as to which region to use for PVC snapshots.
	* Restore: CRD that stores information as to what contents of a backup should be restored.
	* BackupController: Controller inside the server that manages the CRDs (backups / restores / schedules) and processes Kubernetes API calls.

* Example Backup Storage File looks like this

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: <BUCKET_NAME>
  config:
    region: <YOUR_REGION>
    s3ForcePathStyle: "true"
    s3Url: <URL_ACCESS_POINT>
```

### Disaster Recovery Scenario:

Now deploy a sample Nginx application with a specific namespace and we will check the Disaster-Management scenerio by taking a backup of the application simulate a disaster and show how we can restore it by using velero.

Using Kubectl create an nginx deployment.

```
kubectl apply -f nginx/base.yaml
```
After the deployment is created, use velero to take backup the namespace by using the following command:

```
velero backup create <YOUR_BACKUP_NAME> --include-namespaces <YOUR_NAMESPACES> 
```
#### Schedule backup:
We can also take the schedule backups using command

```
velero schedule create <YOUR_BACKUP_NAME> --schedule="0 10 * * *" --include-namespaces <YOUR_NAMESPACES>
(or)
# This will take the backup of the entire cluster every 10 minutes, you can include the specific namespaces, you can also give an expiration time for the backup to stay in the bucket.
velero schedule create sample-backup \
    --schedule="0 10 * * *" \
    --include-resources '*' \
    --include-namespaces '*' \
    --include-cluster-resources=true \
    --ttl 24h 

```
##### Note : Here, The default TTL is 30 days (720 hours); you can use the `--ttl` flag to change this as necessary.

Now we simulate the disaster, Assuming someone deleted the nginx namespace. Go ahead and delete the namespace using the following command.

```
kubectl delete namespace nginx-example
#check if still the resource exists or not by using the following command
kubectl get deployments --namespace nginx example
```
Finally,we will use the Velero to restore the namespace by following command:

```
velero restore create --from-backup <YOUR_BACKUP_NAME>
(or)
velero restore create --from-backup <YOUR-BACKUP-NAME> \
    --include-namespaces '*' \
    --include-resources '*' \
    --include-cluster-resources true 
```
If you have any troubleshooting issue while restore, refer to this [link](https://velero.io/docs/v0.11.0/debugging-restores/)

If your backups or restores stuck in New phase,refer to the "General" section in this [link](https://velero.io/docs/v0.11.0/debugging-install/)

#### What is [Restic](https://restic.net/)

From 0.9 version Restic support, Velero now supports taking backup of almost any type of Kubernetes volume regardless of the underlying storage provider.

Setting up Velero with restic:

 ```kubectl apply -f config/minio/30-restic-daemonset.yaml```
 
 This [Document](https://velero.io/docs/v0.11.0/restic/) will give you a complete idea of What is Restic, How it works, How to setup the Restic on your cluster, What type of backups it can take, How to restore from the backup's. 
 
 For Troubleshooting issues with the Restic refer this [link](https://velero.io/docs/v0.11.0/restic/#troubleshooting)

### How to take the PV backup and Restore's using Velero with the Restic support. 

In this case while you are trying to backup your PV, we have to annotate each pod that has volume to be backup. The document consists of annotating the pods using 'kubectl' try to annotate using Helm charts.

##### NOTE:Velero with Restic works in an incremental fashion. If you are restoring into the existing namespace, It will first check whether the namespace is present or not. If not creates the new namespace and add the resources. If namespace already exists, It will not override the resources. It just add the missing resources into the namespace.

Backup:
``` 
velero create backup <BACKUP_NAME> --include-namespaces <YOUR_NAMESPACE> --snapshot-volumes 
(or)
velero schedule create <BACKUP_NAME> \
    --schedule="0 10 * * *" \
    --include-resources '*' \
    --include-namespaces '*' \
    --include-cluster-resources=true \
    --ttl 24h \
	--snapshot-volumes true
```
Restore:
``` 
velero create restore --from-backup <BACKUP_NAME> --restore-volumes 
(or)
velero restore create --from-backup <YOUR-BACKUP-NAME> \
    --include-namespaces '*' \
    --include-resources '*' \
    --include-cluster-resources true \
    --restore-volumes true
```

### We can also use "app names" for taking backup/restores. 
* Backup:

`velero backup create <backup-name> --selector app=<app-name>` 

* Restore:

`velero restore create --from-backup <backup-name>`

## Velero commands

* To list the backup `velero backup get`
* To list the restores `velero restore get`
* To describe the backup details `velero backup describe <YOUR_BACKUP_NAME>`
* To describe the restore details `velero restore describe <YOUR_RESTORE_NAME>`
* To see the backup logs `velero backup describe <YOUR_BACKUP_NAME>`
* To see the restore logs `velero restore describe <YOUR_RESTORE_NAME>`

## Cluster Migration with Velero

With Velero one can backup the important resources in one cluster and restore it in another cluster by making both clusters pointing to the same storage location. 
This [Document](https://velero.io/docs/v0.11.0/migration-case/) gives you a clear idea. 
