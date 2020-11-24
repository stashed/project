# Use Custom Sidecar


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Use Custom Sidecar](#use-custom-sidecar)
  - [Motivation](#motivation)
    - [Goal](#goal)
    - [Non-goal](#non-goal)
  - [Proposal](#proposal)
  - [Backward Compatibility](#backward-compatibility)
  - [Possible Problems and Solution](#possible-problems-and-solution)
      - [What if the user provide `target.sidecar` during Function-Task model?](#what-if-the-user-provide-targetsidecar-during-function-task-model)
      - [What if the provided Function does not exist?](#what-if-the-provided-function-does-not-exist)
      - [What if the provided Function does not have a valid container definition?](#what-if-the-provided-function-does-not-have-a-valid-container-definition)

<!-- /code_chunk_output -->

## Motivation

Currently, Stash inject a predefined sidecar for backup in sidecar-model. This sidecar takes backup of the directories pointed by the respective `BackupConfiguration` or `BackupBatch`. However, some applications may need to execute some complex logics before taking the backup. Those complex logics can not be executed by the existing hooks. Also, existing Function-Task model is also not applicable for such application if it needs access to the application's volumes.

For example, for MariaDB we can take two different types of backup. Full-backup that takes backup of the entire database and incremental-backup that takes backup of binary log since last backup. The sidecar must be able to take full-backup or incremental-backup on different sessions. This behavior can not be achieved with current pre-defined sidecar.

### Goal

The goal of this proposal can be summarized as bellow:

- Allow injecting a custom sidecar in sidecar model.
- Ensure backward compatibility with existing model.

### Non-goal

These are not the goal of this proposal:

- Allow injecting multiple sidecars.
- Allow injecting third party sidecar (i.e. `fluentd`, `istio MTLS` etc.) for other than backup purpose.

## Proposal

In Function-Task model, we resolve Function and Task to create the backup job. We can follow the similar approach here. User can provide the custom-sidecar definition as a Function and Stash can resolve the function and inject the resolved container as sidecar.

We can add a sidecar section in target specification that takes a Function name an some optional parameters. The YAML of a sample `BackupConfiguration` with custom sidecar is shown below:

```yaml
apiVersion: stash.appscode.com/v1beta2
kind: BackupConfiguration
metadata:
  name: mariadb-backup
  namespace: demo
spec:
  driver: Restic
  repository:
  - name: gcs-repo
  target:
    alias: sample-mariadb
    ref:
      apiVersion: apps/v1
      kind: StatefulSet
      name: sample-mariadb
    sidecar:
      functionName: maria-backup
      params:
        a: b
        c: d
  retentionPolicy:
    name: 'keep-last-5'
    keepLast: 5
    prune: true
```

Here,
- **spec.target.sidecar.functionName** specifies the name of the Function that contains the container definition of the custom sidecar.
- **spec.target.sidecar.params** are optional parameters (key value pair) that can be passed as arguments to the custom sidecar.

## Backward Compatibility

Stash will inject the custom sidecar only if `target.sidecar` section is provided. Otherwise, Stash will inject the pre-defined `stash` sidecar. So, it should be backward compatible with existing behavior. No additional efforts are required.

## Possible Problems and Solution

We may face the following issues for this features.

####  What if the user provide `target.sidecar` during Function-Task model?

In this case, Stash should forbid providing `target.sidecar` if `spec.task` section is specified. We can enforce it from the validator.

#### What if the provided Function does not exist?

In this case, Stash should forbid creating `BackupConfiguration` with a Function name that does not exist yet using the validating webhook.

#### What if the provided Function does not have a valid container definition?

In this case, Stash should provide a clear error message and write event against BackupConfiguration denoting that it has failed to resolve the sidecar.
