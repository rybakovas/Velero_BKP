# Velero_BKP
Bakup solution for Openshift based on Velero and MinIO

# Running Backup & Restore on an OpenShift environment using Velero and Minio



The goal of this Documentation is to provide a step-by-step tutorial on how to set up, backup and restore an application running on CodeReady Containers OpenShift, using Velero for Backup and Restore and Minio as S3-like Object Storage.

From velero [documentation](https://velero.io/docs/v1.0.0/restic/) we find that velero allows the user to take snapshots of persistent  volumes as part of the backups if you are using one of the supported  cloud providers’ block storage offerings (Amazon EBS Volumes, Azure  Managed Disks, Google Persistent Disks).

But, if you are using a volume type that does not have a native  snapshot concept, velero integrates with restic to enable it. The  downside is that restic does not support *hostPath* volume type, so to make this tutorial possible, we chose to use the *local* volume type.



## Minio Install

Do the git clone of Velero from official address

```
git clone https://github.com/vmware-tanzu/velero.git
```

Go to path where is the Minio deployment yaml file

```
cd velero/examples/minio/
```

Login into Openshift by command line

```
oc login --username=myuser --password=mypass --server=https://<OCPaddress>:6443
```

Create Velero project

```
oc new-project velero
oc project velero
```

Deploy Minio and get the address to access your buckets 

```
oc apply -f 00-minio-deployment.yaml
oc expose svc minio
oc get route minio
```

Create Credential File to Velero access Minio

```
vi credentials-velero

[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
```

## Velero Install

Go to the path where you did the Git Clone and access velero folder, from there you will create the command line to install the Velero into the OCP Cluster and the velero client in your linux

```
./velero install --provider aws --use-restic --plugins velero/velero-plugin-for-aws:v1.0.0 --bucket velero --secret-file ./credentials-velero --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```

Give the privileges for Velero and Restic 

```
oc adm policy add-scc-to-user privileged -z velero -n velero

oc patch ds/restic --namespace velero --type json -p '[{"op":"add","path":"/spec/template/spec/containers/0/securityContext","value": { "privileged": true}}]'
```

Done the Velero is Install.

## Running Backup

##### Annotate each pod that you want to backup

Each pod with a persistent volume needs to be annotated in order for restic to detect them and back them up. For that run the command below:

```
oc -n YOUR_POD_NAMESPACE annotate pod/YOUR_POD_NAME backup.velero.io/backup-volumes=YOUR_VOLUME_NAME_1,YOUR_VOLUME_NAME_2,...
```

where YOUR_VOLUME_NAME_1, ... is the name of the volume found in the  mysql-deploymet.yaml and wordpress-deployment.yaml for the respective  pods. i.e.:

```
volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
```

##### Run backup command

```
velero backup create NAME OPTIONS
```

For version 1.5 or higher you can include the option  `--default-volumes-to-restic` into backup creation command, with this option the backup will use Restric by default without the necessity to include the pod annotate

```
velero backup create NAME OPTIONS --default-volumes-to-restic
```



## Running Restore

##### Run restore command

To restore the application from backup you need to run the command below:

```
velero restore create --from-backup BACKUP_NAME OPTIONS...
```

or you can flag some schedule, in this case Velero will use the last backup to create the restore:

```
velero restore create --from-schedule SCHEDULE_NAME OPTIONS...
```

useful option is `--include-namespaces <NAMESPACE1,NAMESPACE2,...>` with is option you can select what namespace(project) do you want provide the restore.

## Create a schedule

```
velero create schedule nginx-namespace --schedule="0 11 * * *" --include-namespaces <NAMESPACE>
```

## Assumption

In this test we had try to do the backup and restore with 2 types of StorageClass base in the Provisioner bellow :
`cluster.local/nfs-client-provisioner (Retain Policy - Retain)`
and
`kubernetes.io/vsphere-volume (Retain Policy - Delete)`

For Volumes mounted in NFS storage class the solution doesn't work to do the Backups / Restores for PV / PVCs 

For Volumes mounted in  vsphere-volume the backup and restore solution works using restric as snapshot provider

