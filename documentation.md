# Disaster Management with Heptio-Velero 

### A Quick info on Velero

[Velero](https://velero.io) (formerly Heptio Ark) gives you tools to back up and restore your Kubernetes cluster resources. It also takes snapshots of your cluster's Persistent Volumes using your cloud provider's block storage snapshot features, and can then restore your cluster's objects and Persistent Volumes to a previous state using [Velero Restic Setup](https://velero.io/docs/v1.0.0/restic/). Velero can perform full backups of the entire cluster or can limit what it backs up by namespaces or resource types. Backups can be run manually or on a scheduled basis

For detail explanation of how Velero works follow the [link](https://velero.io/docs/v1.0.0/about/)

#### Reasons to Take Backups of Kubernetes Clusters

* To recover from
   * Kubernetes namespace deletion.
   * Quick recovery when a cluster is lost or inaccessible.
   * When a bug cleaned out your PV and all your data is lost.
* To back up a cluster prior to upgrading it in case of a catastrophic failure that requires recreation of the cluster.

## Prerequisites

* Access to a Kubernetes cluster, version 1.7 or later. Version 1.7.5 or later is required to run Velero.
* kubectl installed

#### Setup Velero on your cluster

This [link](https://velero.io/docs/v1.0.0/ibm-config/) explains how to setup Velero on IBM Cloud with IBM Cloud Object Storage as Velero's storage destination. 

* Download an official [release](https://github.com/heptio/velero/releases) of Velero
  * Run `velero-extracts.sh` which extracts the files needed for the IBM cloud. 
  * To Upgrade from version v0.11.0 to v1.0.0, follows the Part 2 section of this [link](https://velero.io/docs/v1.0.0/upgrade-to-1.0/) 
* Create your [COS instance](https://cloud.ibm.com/docs/services/cloud-object-storage/basics?topic=cloud-object-storage-provision#creating-a-new-resource-instance).
* Create an [S3 bucket](https://cloud.ibm.com/docs/services/cloud-object-storage?topic=cloud-object-storage-getting-started#create-buckets)
* Define a [service](https://cloud.ibm.com/docs/services/cloud-object-storage/iam?topic=cloud-object-storage-service-credentials#service-credentials) that can store data in the bucket. Follow each and every step listed in the official document [here](https://velero.io/docs/v1.0.0/ibm-config/)  
* Configure and start the Velero server. Check the "Credentials and Configuration" section in this [link](https://velero.io/docs/v1.0.0/ibm-config/)
  * For easy installation follow the steps below.
    
	`
	#To deploy the Velero pre-requisities
	kubectl apply -f config/common/00-prereqs.yaml
	#To create a secret for your "Cloud Credentials" in the directory of the credentials file you just created, run:
	kubectl create secret generic cloud-credentials \
    --namespace <VELERO_NAMESPACE> \
    --from-file cloud=credentials-velero
	#After configuring your bucket with the <BUCKET_NAME>,<REGION> and <URL_ACCESS_POINT> in the Backup Storage File as in the example shown below, start the Velero server 
	kubectl apply -f config/ibm/05-backupstoragelocation.yaml
    kubectl apply -f config/ibm/10-deployment.yaml
	`
	(or) 
	
	Simply run the `cloudsecret.sh` to set the secret for the cloud credentials.
	
	`sh ./cloudsecret.sh <VELERO_NAMESPACE>
	 ex:./cloudsecret.sh velero
	`
	After configuring your bucket with the <BUCKET_NAME>,<REGION> and <URL_ACCESS_POINT> in the Backup Storage File, start the Velero server by running 
	
	`velero-setup.sh` 
	
* Example Backup Storage File looks like this

`yaml
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
`
  * It is important to know what's going on behind the scenes before apply the pre-req's in the above link. 
    * Backup: CRD that stores information such as creation date, which namespaces should be included, which PVCs are attached, and so on.
	* BackupLocation: CRD that stores information such as which region and object storage should be used to store backups.
	* SnapshotLocation: CRD that stores information as to which region to use for PVC snapshots.
	* Restore: CRD that stores information as to what contents of a backup should be restored.
	* BackupController: Controller inside the server that manages the CRDs (backups / restores / schedules) and processes Kubernetes API calls.



### Disaster Recovery Scenario:

Now deploy a sample Nginx application with a specific namespace and we will check the Disaster-Management scenerio by taking a backup of the application simulate a disaster and show how we can restore it by using velero.

Using Kubectl create an nginx deployment.

```
cd $HOME/heptio-velero-config
kubectl apply -f config/nginx-app/base.yaml
```
After the deployment is created, use velero to take backup the namespace by using the following command:

```
velero backup create <YOUR_BACKUP_NAME> --include-namespaces <YOUR_NAMESPACES> 
#In our example
velero backup create samplebackup --include-namespaces nginx-example

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
#In our example	
velero schedule create samplebackup --schedule="0 10 * * *" --include-namespaces nginx-example  
```
#### Note : The default TTL is 30 days (720 hours); you can use the `--ttl` flag to change this as necessary

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
#In our example
velero restore create --from-backup nginx-example	
```
If you have an issue during the restore, refer to this [link](https://velero.io/docs/v1.0/debugging-restores/)

If your backups or restores stuck in New phase,refer to the "General" section in this [link](https://velero.io/docs/v0.11.0/debugging-install/)

### What is [Restic](https://restic.net/)

From 0.9 version Restic support, Velero now supports taking backup of almost any type of Kubernetes volume regardless of the underlying storage provider.

 This [Document](https://velero.io/docs/v1.0.0/restic/) will cover the following topics: 
  * What is Restic?
  * How to setup the Restic on your cluster?
    * For easy installation. Check into the `config/minio` directory which you get from the extracts and follow the below steps
	  * ```
	     cd $HOME/heptio-velero-config
		 kubectl apply -f config/minio/30-restic-daemonset.yaml
	    ```
  * How backup and restore work with restic? 
  
For Troubleshooting issues with the Restic refer this [link](https://velero.io/docs/v0.11.0/restic/#troubleshooting)

### How to take the PV backup and Restore's using Velero with the Restic support. 

In this case while you are trying to backup your PV, we have to annotate each pod that has volume to be backup. Though the example in document show using `kubectl` to set the annotations, you should add them via a helm chart or some other automation method.

#### NOTE: Velero with Restic works in an incremental fashion. If you are restoring into the existing namespace, it will first check whether the namespace is present or not. If not, it creates the new namespace and adds the resources. If namespace already exists, It will not override the resources. It will just add the missing resources into the namespace.

### Now deploy a sample Nginx application with a Persistent Volume in a specific namespace and we will check the Disaster-Management scenerio by taking a backup of the volume and restore into the same namespace by simulating disaster.
* Before you run the nginx example, in file config/nginx-app/with-pv.yaml:
   * Replace <YOUR_STORAGE_CLASS_NAME> with your StorageClass name.
```
cd $HOME/heptio-velero-config
kubectl apply -f config/nginx-app/with-pv.yaml
```

After the deployment is created, use velero to take backup the namespace by using the following command:

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
#In our example
velero backup create samplebackup --include-namespaces nginx-example	
(or)
velero schedule create samplebackup \
    --schedule="0 10 * * *" \
    --include-namespaces 'nginx-example' \
    --ttl 24h \
    --snapshot-volumes true
	
```
Now we simulate the disaster, Assuming someone deleted the nginx namespace. Go ahead and delete the namespace using the following command.
```
kubectl delete namespace nginx-example
#check if still the resource exists or not by using the following command
kubectl get deployments --namespace nginx example
```
Finally,we will use the Velero to restore the namespace by following command:

Restore:
``` 
velero create restore --from-backup <BACKUP_NAME> --restore-volumes 
(or)
velero restore create --from-backup <YOUR-BACKUP-NAME> \
    --include-namespaces '*' \
    --include-resources '*' \
    --include-cluster-resources true \
    --restore-volumes true
	
#In our example
velero restore create --from-backup sample-backup
(or)
velero restore create --from-backup sample-backup \
    --include-namespaces 'nginx-example' \
    --restore-volumes true
```

### We can also use `--selector` flag to select labels for taking backup/restores. 
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
See [migration cases](https://velero.io/docs/v1.0.0/migration-case/) for details on how to do this.
