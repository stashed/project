# Sharded MongoDB Backup

## Prerequisites

- Install [KubeDB](https://kubedb.com/docs/latest/setup/).
- Install [Stash Enterprise](https://stash.run/docs/latest/setup/install/enterprise/).
- Install [MongoDB Addon](https://stash.run/docs/latest/addons/mongodb/setup/install/) for Stash.
- A storage bucket. Here, we are going to use a Google Cloud Storage bucket.

## Deploy MongoDB

- Deploy MongoDB

```bash
kubectl apply -f ./mongo.yaml
```

**Insert sample data:**

- Retrieve authentication info

```bash
# username
$ kubectl get secrets -n demo mg-sh-auth -o jsonpath='{.data.\username}' | base64 -d
root

# password
$ kubectl get secrets -n demo mg-sh-auth -o jsonpath='{.data.\password}' | base64 -d
'k,xzFjbX62u2L-Rc'
```

- Exec into the database pod

```bash
$ kubectl exec -it -n demo mg-sh-mongos-0 -- mongo admin -u root -p k,xzFjbX62u2L-Rc
```

- List available databases

```bash
$ show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
```

- Create a new database

```bash
$ use kubedb
switched to db kubedb
```

- Insert data

```bash
$ db.demo.insert({"kubedb": "shard demo"});
WriteResult({ "nInserted" : 1 })
```

- Verify that the data has been inserted

```bash
$ db.demo.find()
{ "_id" : ObjectId("5f9aa70511be4af570bda50d"), "kubedb" : "shard demo" }
```

- List available databases

```bash
> show dbs
admin   0.000GB
config  0.001GB
kubedb  0.000GB
```

## Backup

- Verify that AppBinding has been created by KubeDB

```bash
$ kubectl get appbindings -n demo mg-sh
NAME    TYPE                 VERSION   AGE
mg-sh   kubedb.com/mongodb   4.2.3     7m19s
```

- Create Secret

```bash
$ kubectl create secret generic -n demo gcs-secret \
      --from-file=./RESTIC_PASSWORD \
      --from-file=./GOOGLE_PROJECT_ID \
      --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
```

- Create Repository

```bash
$ kubectl apply -f ./repository.yaml
```

- Create BackupConfiguration

```bash
$ kubectl apply -f ./backupconfiguration.yaml
```

- Verify CronJob

```bash
$ kubectl get cronjob -n demo
NAME                        SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stash-backup-mg-sh-backup   */5 * * * *   False     0        <none>          23s
```

- Trigger instant backup or wait till next scheduled slot.

```bash
$ kubectl apply -f ./backupsession.yaml
```

- Verify backup has succeeded

```bash
$ kubectl get backupsession -n demo
NAME                      INVOKER-TYPE          INVOKER-NAME   PHASE       AGE
mg-sh-backup-1603986911   BackupConfiguration   mg-sh-backup   Succeeded   3m13s
```

## Restore

- Pause Backup

```bash
$ kubectl patch backupconfiguration -n demo mg-sh-backup --type="merge" --patch='{"spec": {"paused": true}}'
```

- Verify CronJob paused

```bash
$ kubectl get cronjob -n demo
NAME                        SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stash-backup-mg-sh-backup   */5 * * * *   True      0        4m33s           4h28m
```

- Simulate disaster (delete some data)

```bash
$ kubectl exec -it -n demo mg-sh-mongos-0 -- mongo admin -u root -p 'tcLnX)ViIZOKBrod'
MongoDB shell version v4.2.3
...
mongos> show dbs
admin   0.000GB
config  0.002GB
kubedb  0.000GB

# switch to kubedb database
mongos> use kubedb
switched to db kubedb

# delete kubedb database
mongos> db.dropDatabase();
{
        "dropped" : "kubedb",
        "ok" : 1,
        "operationTime" : Timestamp(1603987229, 6),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1603987229, 6),
                "signature" : {
                        "hash" : BinData(0,"dGAIm/30GggQUdqtN2iB8SEf2HI="),
                        "keyId" : NumberLong("6889065897118400524")
                }
        }
}

# verify that the database has been deleted
mongos> show dbs
admin   0.000GB
config  0.002GB

mongos> exit
bye
```

- Create RestoreSession

```bash
$ kubectl apply -f ./restoresession.yaml
restoresession.stash.appscode.com/sample-mongo-restore created
```

- Wait for restore to success

```bash
$ kubectl get restoresession -n demo -w
NAME            REPOSITORY   PHASE     AGE
mg-sh-restore   gcs-repo     Running   8s
mg-sh-restore   gcs-repo     Running   8s
mg-sh-restore   gcs-repo     Running   27s
mg-sh-restore   gcs-repo     Succeeded   27s
mg-sh-restore   gcs-repo     Succeeded   27s
```

- Verify restored data

```bash
$ kubectl exec -it -n demo mg-sh-mongos-0 -- mongo admin -u root -p 'tcLnX)ViIZOKBrod'
MongoDB shell version v4.2.3
...
# verify the database has been restored
mongos> show dbs
admin   0.000GB
config  0.002GB
kubedb  0.000GB

# switch to the restored database
mongos> use kubedb
switched to db kubedb

# verify that the data has been restoed
mongos> db.demo.find();
{ "_id" : ObjectId("5f9ae52f9cca8d9d10cf410e"), "kubedb" : "shard demo" }
mongos> exit
bye
```
