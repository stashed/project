# Support Snapshot Listing for Local Backend (NFS)

<!-- code_chunk_output -->

- [Support Snapshot Listing for Local Backend (NFS)](#support-snapshot-listing-for-local-backend-nfs)
  - [Motivation](#motivation)
    - [Goal](#goal)
  - [Proposed Models](#proposed-models)
    - [Run on-demand pod mounting the `local` backend during Snapshot listing](#run-on-demand-pod-mounting-the-local-backend-during-snapshot-listing)
    - [Always runs a Deployment mounting the `local` backend](#always-runs-a-deployment-mounting-the-local-backend)
  - [Possible Issues](#possible-issues)
    - [Container Resources](#container-resources)
    - [PSP Enabled Cluster](#psp-enabled-cluster)
  - [Conclusion](#conclusion)

<!-- /code_chunk_output -->

## Motivation

Viewing Snapshots of a particular Repository is necessary for restoring purposes or to be sure that the backup has created some Snapshots.

Stash provides Snapshot listing facility through an Extended API Server(EAS). This EAS runs as a part of the Stash operator. When a user request for Snapshots of a Repository, Stash operator directly connect with the backend using `restic` and return the Snapshot information stored in the backend. This method works well for cloud backends.

However, there is some implication for `local` backend. The `local` backend is mounted inside the backup sidecar/job as a volume. So, the Stash operator can't directly access those volumes. In this case, the Stash operator exec into the container that mounts the `local` volume and lists the Snapshot from inside that container. This method has the following problems:

1. The container which has mounted the `local` backend must be running during listing the Snapshot.
2. This method won't work for job-model because the backup jobs are ephemeral. They terminate the containers once the backup is completed.
3. If the user wants to restore it in a different cluster, this method won't work. The backup sidecar won't be running there.

### Goal

The goal of this KEP can be summarized as the followings:

1. Propose some models to solve/mitigate the above issue.
2. Discuss the possible issues that might arise for the proposed models.

## Proposed Models

Here, we are going to propose two different possible solutions for the problem and discuss the pros and cons of each solution.

### Run on-demand pod mounting the `local` backend during Snapshot listing

We can run a pod mounting the `local` backend in the same node as the respective backup sidecar/job. Then, the Stash operator can exec into the pod and list the desired Snapshots.

**Pros:**

- Since the pod runs on-demand, it has no overhead of running a dedicated workload only for Snapshot listing purposes.

**Cons:**

- Starting a pod, then exec into it and finally terminating it requires some time. This may not fit inside the time limit for the list/get API. So, Snapshot listing may encounter request timeout.
- Since the Stash operator has permission to create pod in any namespace, a user can use this capability to list the Snapshots from an unauthorized namespace.

### Always runs a Deployment mounting the `local` backend

We can run a Deployment in each namespaces that has a Repository with `local` volume (especially NFS or similar network volumes). The Deployment will mount the local volumes used in the Repositories of this namespace. Stash operator can exec into the Deployment pod and list the desired Snapshot.

**Pros:**

- Since the Deployment is always running, this will solve the request timeout issue.
- Running Deployment into the same namespace as the Repository will solve the Snapshot listing issue from an unauthorized namespace.

**Cons:**

- An extra overhead for running those Deployments.

Since the Deployments will stay idle for most of the time and will use resource only when the user requests to list Snapshots, this overhead shouldn't be an issue.

## Possible Issues

There might be some issues with the proposed models. They are acknowledged below.

### Container Resources

Some clusters do not allow creating a workload without specifying the `resources` section of the container. Since the pod/Deployment will be created by the Stash operator itself, the user won't have control over this field.

One possible solution to this problem could be always creating the pod/Deployment with some minimum amount of `resources`. This could be `50m` for CPU and `128M` for Memory.

### PSP Enabled Cluster

We might need to set a default PSP for PSP enabled clusters. Since the pod/Deployment will do just read-only operation, recently introduced `baseline` PSP should be sufficient.

## Conclusion

From the above discussion, we believe the second method is more preferable than the first one. We are open to any idea that may solve this problem more securely and efficiently.
