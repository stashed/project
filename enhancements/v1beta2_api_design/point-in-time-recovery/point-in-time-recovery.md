# Point-in-Time Recovery


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Point-in-Time Recovery](#point-in-time-recovery)
  - [Motivation](#motivation)
    - [Goal](#goal)
  - [Proposal](#proposal)
    - [Why keep meta file in the backend repository?](#why-keep-meta-file-in-the-backend-repository)
  - [How it going to work](#how-it-going-to-work)
  - [Backward Compatibility](#backward-compatibility)
  - [Possible Problems and Solution](#possible-problems-and-solution)
      - [Who will update the meta file for concurrent backup?](#who-will-update-the-meta-file-for-concurrent-backup)

<!-- /code_chunk_output -->

## Motivation

Currently, if a user want to rollback her application to a certain point in time, she first have to list all the snapshots from the respective Repository. Then, she has to find the snapshot that was taken at/before her desired time. Then, she has to point the snapshot in `RestoreSession` CRD so that Stash can restore the desired state.

This manual process does not provide the best experience that Stash can possibly provide in such use-case. Ideally, the user should only need to provide her desired timestamp and Stash itself should figure out the appropriate snapshot to restore.

This proposal outlines how we can implement point-in-time recovery where user will only provide the target time and Stash will automatically restore the appropriate snapshot.

### Goal

The goal of this proposal can be summarized as bellow:

- Propose possible ways of implementing point-in-time recovery.

## Proposal

We should introduce a `pitr` field in the `rules` section of `RestoreSession` CRD. The user will will provide her target time in `pitr.targetTime` field.

Here, is the sample YAML of proposed `RestoreSession`,

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: RestoreSession
metadata:
  name: sample-mysql-restore
  namespace: demo
  labels:
    kubedb.com/kind: MySQL
spec:
  task:
    name: mysql-restore-8.0.14-v2
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: appcatalog.appscode.com/v1alpha1
      kind: AppBinding
      name: restored-mysql
  rules:
  - paths:
    - /source/data/
    pitr:
      targetTime: "2019-09-27T05:07:34Z"
```

Now, Stash have to figure out the appropriate snapshot for this target time. In order to do that, Stash can list all the snapshots, check their creation timestamp and find out the latest that meet this targeted time. However, this method is not very efficient because `restic` requires lots of time to list the snapshots.

In order to solve this problem, we can keep an meta file (maybe `meta.yaml`) in the root of the repository that will hold the information about all the snapshots of this repository.

A sample `meta.yaml` file is given below,

```yaml
repoInfo: # maybe Stash CLI can use this info to re-construct Repository CRDs in a new cluster
  name: gcs-repo
  namespace: demo
  backend:
    gcs:
      bucket: stash-backup
      prefix: demo
    storageSecretName: gcs-secret
  retentionPolicy:
    name: "keep-last-5"
    keepLast: 5
    prune: true
snapshots:
- name: gcs-repo-b54ee4a0
  uid: b54ee4a0e9c9084696dc976f125c4fd0e6b1a31abfd82cfc857b3bc9e559fa2f
  creationTimestamp: "2020-07-24T17:41:31Z"
  hostname: app
  username: "stash-demo-2lsowls"
  userID: 2000
  groupID: 2000
  paths:
  - /var/lib/html
- name: gcs-repo-b54ee4a0
  uid: aedade4a0e9c9084696dc976f125c4fd0e6b1a31abfd82cfc857b3bc9e559fa2f
  creationTimestamp: "2020-07-23T17:41:31Z"
  hostname: app
  username: "stash-demo-2lsowls"
  userID: 2000
  groupID: 2000
  paths:
  - /var/lib/html
- name: gcs-repo-b54ee4a0
  uid: 3s3sc9084696dc976f125c4fd0e6b1a31abfd82cfc857b3bc9e559fa2f
  creationTimestamp: "2020-07-25T17:41:31Z"
  hostname: app
  username: "stash-demo-2lsowls"
  userID: 2000
  groupID: 2000
  paths:
  - /var/lib/html
```

### Why keep meta file in the backend repository?

The meta file will serve the following purposes:

- **Point-in-Time Recovery:** The meta file will contain the information about all the snapshot on the repository. So, during point-in-time recovery Stash can only read the meta file and figure out the appropriate snapshot for the targeted time.
- **List Snapshots:** Currently, listing snapshot is done via the EAS (Extended API Server). When user list Snapshots, the EAS runs `restic list` command and construct the Snapshot from the output. Since, `restic list` command is very time consuming, this often result timeout. If we keep meta file in the repository, then the EAS can only read the meta file and return the snapshot list. It will be lot more faster and will solve the timeout issue.
- **GDPR:** For GDPR feature, we will have to restore a snapshot, remove the unwanted data from it. Then, we have to take another snapshot of the new data. Finally, we have to remove the old snapshot. Since, we are taking new snapshot, the original snapshot date will get lost. Hence, it will be impossible for the user to tell which snapshot contains data of a particular time. Instead, if we keep meta file in the backend repository, we can just update the uid of the old Snapshot entry to point to the new snapshot. So, original snapshot time will be preserved.
- **Repository restoration:** If we keep the original repository information in the meta file, Stash CLI will be able to re-construct the Repository definition from it. This will help the user to restore data in a new cluster. She will run a Stash CLI command to restore the Repositories and then he can create a RestoreSession pointing to that Repository.

## How it going to work

At first, Stash will read the meta file from the backend. Then, it will figure out the latest snapshot that satisfy the targeted time. Finally, it will run restore with the particular snapshot.

## Backward Compatibility

Since, `RestoreSession` is one-time object, we don't need any extra work to keep it backward compatible.

## Possible Problems and Solution

We may face the following issues for this features.

#### Who will update the meta file for concurrent backup?

Stash allows to backup multiple host in the same repository concurrently. As a result, we have to control who will update the meta file. Otherwise, it can get corrupted. In this case, we can follow the repo initialization or retention policy applying method. Stash operator assign which backup executor will initialize the repository and which executor will apply retention policy. We can do same for meta file updating. The process that apply the retention policy can also update the meta file because after applying retention policy some snapshot get removed. We have to remove them from the meta file too.
