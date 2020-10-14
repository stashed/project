# Execute hooks in custom container


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Execute hooks in custom container](#execute-hooks-in-custom-container)
  - [Motivation](#motivation)
    - [Goal](#goal)
  - [Proposal](#proposal)
  - [Backward Compatibility](#backward-compatibility)
  - [Possible Problems and Solution](#possible-problems-and-solution)
      - [What if the provided Function does not exist?](#what-if-the-provided-function-does-not-exist)
      - [What if the provided Function does not have a valid container definition?](#what-if-the-provided-function-does-not-have-a-valid-container-definition)
      - [How should we cleanup the hook executor job?](#how-should-we-cleanup-the-hook-executor-job)
      - [What if a hook execution fail? Should we retry?](#what-if-a-hook-execution-fail-should-we-retry)

<!-- /code_chunk_output -->

## Motivation

Currently, Stash executes the global hooks inside the operator. As a result, the functionality become limited. For example, to execute some complex command, some application specific binary may required. So, it is not practical for the operator to meet such use cases. In this case, a dedicated hook executor will make more sense.

### Goal

The goal of this proposal can be summarized as bellow:

- Allow executing hook in custom container.
- Ensure backward compatibility with existing system.

## Proposal

We can follow the similar approach as Function-Task model. User can provide the hook executor as a Function. Stash will resolve the function and create a job to execute the hook.

The YAML of a sample `BackupConfiguration` with a custom hook executor is shown below:

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupConfiguration
metadata:
  name: backup-hook-demo
  namespace: demo
spec:
  schedule: "*/5 * * * *"
  task:
    name: mysql-backup-8.0.14
  repository:
    name: gcs-repo
  hooks:
    preBackup:
      exec:
      - /bin/sh
      - -c
      - mysql -u root --password=$MYSQL_ROOT_PASSWORD -h sample-mysql.demo.svc -p 3366 -e "SET GLOBAL super_read_only = ON;"
      executor:
        functionName: mysql-hook-executor
        params:
          a: b
          c: d
  target:
    ref:
      apiVersion: apps/v1
      kind: StatefulSet
      name: sample-sts
  retentionPolicy:
    name: keep-last-5
    keepLast: 5
    prune: true
```

Here,
- **spec.hooks.preBackup.executor.functionName** specifies the name of the Function that will be used to create hook executor job.
- **spec.hooks.preBackup.executor.params** are optional parameters (key value pair) that can be passed as arguments to the hook executor.

## Backward Compatibility

Stash will create the executor job only if `executor` section is provided in a hook. Otherwise, Stash will execute the hook as it executes now. So, it should be backward compatible with existing behavior. No additional efforts are required.

## Possible Problems and Solution

We may face the following issues for this features.

#### What if the provided Function does not exist?

In this case, Stash should forbid creating `BackupConfiguration` with a Function name that does not exist yet using the validating webhook.

#### What if the provided Function does not have a valid container definition?

In this case, Stash should provide a clear error message and write event against BackupConfiguration denoting that it has failed to resolve the hook executor.

#### How should we cleanup the hook executor job?

The respective BackupSession will be the owner of this Job. So, when the BackupSession gets cleaned up according to `backupHistoryLimit`, this job will be garbage collected.

#### What if a hook execution fail? Should we retry?

If a hook execution fail for first time, it will likely to fail in the subsequent attempts. So, there is no need to retry.
