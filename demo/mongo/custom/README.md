# MongoDB Demo

## Deploy MongoDB

- Deploy MongoDB

```bash
kubectl apply -f ./mongo.yaml
```

**Insert sample data:**

- Exec into the database pod

```bash
$ kubectl exec -it -n demo sample-mongo-0 -- /bin/bash
root@sample-mongo-0:/# mongo admin -u admin -p admin123
MongoDB shell version v4.2.3
connecting to: mongodb://127.0.0.1:27017/admin?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("087d7ed2-7f07-41e6-9f89-7fb23d915ba7") }
MongoDB server version: 4.2.3
....
---
```

- List available databases

```bash
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
```

- Create a new database

```bash
> use technology
switched to db technology
```

- Insert data

```bash
> db.cloud.insert({"name":"kubernetes"});
WriteResult({ "nInserted" : 1 })

> db.cloud.insert({"provider":"google"});
WriteResult({ "nInserted" : 1 })
```

- Verify that the data has been inserted

```bash
>db.cloud.find().pretty()
{ "_id" : ObjectId("5f6c2600092bf22d91b7c8c2"), "name" : "kubernetes" }
{ "_id" : ObjectId("5f6c265a092bf22d91b7c8c3"), "provider" : "google" }
```

- List available databases

```bash
> show dbs
admin       0.000GB
config      0.000GB
local       0.000GB
technology  0.000GB
```

## Backup

- Create AppBinding

```bash
$ kubectl apply -f ./appbinding.yaml
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
kubectl apply -f ./repository.yaml
```

- Create BackupConfiguration

```bash
kubectl apply -f ./backupconfiguration.yaml
```

- Verify CronJob

```bash
$ kubectl get cronjob -n demo
NAME                               SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stash-backup-sample-mongo-backup   */5 * * * *   False     0        <none>          7s
5m26s
```

- Trigger instant backup

```bash
kubectl apply -f ./backupsession.yaml
```

- Verify backup has succeeded

```bash
$ kubectl get backupsession -n demo
NAME                             INVOKER-TYPE          INVOKER-NAME          PHASE       AGE
sample-mongo-backup-1600926002   BackupConfiguration   sample-mongo-backup   Succeeded   2m11s
```

## Restore

- Pause Backup

```bash
$ kubectl patch backupconfiguration -n demo sample-mongo-backup --type="merge" --patch='{"spec": {"paused": true}}'
```

- Verify CronJob paused

```bash
$ kubectl get cronjob -n demo
NAME                               SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stash-backup-sample-mongo-backup   */5 * * * *   True      0        119s            14m
```

- Simulate disaster (delete some data)

```bash
$ kubectl exec -it -n demo sample-mongo-0 -- /bin/bash
root@sample-mongo-0:/# mongo -u admin -p admin123
MongoDB shell version v4.2.3
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("382e4a30-8f71-4e4d-908a-b08b171e2248") }
MongoDB server version: 4.2.3
....
---

> show dbs
admin       0.000GB
config      0.000GB
local       0.000GB
technology  0.000GB

> use technology
switched to db technology

> db.dropDatabase();
{ "dropped" : "technology", "ok" : 1 }

> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB

> exit
bye
root@sample-mongo-0:/# exit
exit
```

- Create RestoreSession

```bash
$ kubectl apply -f ./restoresession.yaml
restoresession.stash.appscode.com/sample-mongo-restore created
```

- Wait for restore to success

```bash
$ kubectl get restoresession -n demo -w
NAME                   REPOSITORY   PHASE     AGE
sample-mongo-restore   gcs-repo     Pending   4s
sample-mongo-restore   gcs-repo     Running   5s
sample-mongo-restore   gcs-repo     Succeeded   97s
```

- Verify restored data

```bash
kubectl exec -it -n demo sample-mongo-0 -- /bin/bash
root@sample-mongo-0:/# mongo -u admin -p admin123
MongoDB shell version v4.2.3
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("ee71bd65-a383-47c6-a5aa-60ee7831a267") }
MongoDB server version: 4.2.3
...
---

> show dbs
admin       0.000GB
config      0.000GB
local       0.000GB
technology  0.000GB
> use technology;
switched to db technology
> db.find().pretty();
2020-09-24T06:00:57.126+0000 E  QUERY    [js] uncaught exception: TypeError: db.find is not a function :
@(shell):1:1
> db.cloud.find().pretty();
{ "_id" : ObjectId("5f6c2600092bf22d91b7c8c2"), "name" : "kubernetes" }
{ "_id" : ObjectId("5f6c265a092bf22d91b7c8c3"), "provider" : "google" }
> exit
bye
root@sample-mongo-0:/# exit
exit
```
