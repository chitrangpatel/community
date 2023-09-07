---
status: proposed
title: Enable Step Reusability 
creation-date: '2023-09-07'
last-updated: '2023-09-07'
authors:
- '@chitangpatel'
- '@jerop'
collaborators: []
---

# TEP-0142: Enable Step Reusability 

<!-- toc -->
- [Summary](#summary)
  - [Background](#background)
- [Motivation](#motivation)
- [References](#references)
<!-- /toc -->

## Summary

This TEP proposes to introduce `StepFoo` (**Note**: The naming of this CRD is still under discussion. Once finalized, StepFoo will be replaced by the actual name. Apart from the naming, everything in this doc can be reviewed.) as the smallest, scriptable and reusable unit of work in Tekton. 

The reusable units of work in Tekton, `Tasks`, currently cannot natively execute in the same environment with a shared file system. As such, Tekton users have to choose between reusability and performance. This design doc revisits the components and architecture of Tekton to both enable reusability and optimize performance. 

### Background

`Tasks` are the reusable unit of work in Tekton. A `Task` is made up of a sequence of `Steps`. A `Task` and its `Steps` execute in Kubernetes as a `Pod` made up of a sequence of `Containers`. The `Steps` of a `Task` have access to shared storage and resources, such as `Parameters` and `Workspaces`. `Tasks` are combined in `Pipelines` which are graphs where each node represents a `Task`. The `Tasks` in a `Pipeline` execute in separate `Pods`, thus they need to share data via a `Workspace` that is generally backed by a `Persistent Volume`.

## Motivation

`Steps` are the smallest unit of work but they aren’t reusable because they are specified in `Tasks`. Users end up writing lots of `Tasks` with single `Steps` to make them reusable. In fact, about 77% of `Tasks` in the Tekton Catalog have a single `Step` only. However, these reusable units cannot execute in a shared context – `Pod`. 

When users need to combine `Steps` to execute together, they are forced to choose between reusability and performance. If they prioritize performance over reusability, they would copy and paste the `Steps` of the `Tasks` into a new `Task` to execute in one `Pod` with a shared file system. If they prioritize reusability over performance, they would execute the `Tasks` with single `Steps` in separate `Pods` without a shared file system.

### Case Study

A user relies on [git](https://github.com/tektoncd/catalog/tree/main/task/git-clone/0.9) `Task` to fetch source code from a repo and [kaniko](https://github.com/tektoncd/catalog/blob/main/task/kaniko/0.6/kaniko.yaml) `Task` to build and push an image. 

If they copy and paste the `Steps` of the `Tasks` into a single `Task` to execute in one `Pod`, then the source code will be fetched to a local disk which will be used to build the image. This is performant but hurts reusability.

If they execute the `Tasks` in separate `Pods`, the source code will be fetched to a persistent disk which will be used to build the image. This maintains the well-factored reuse but incurs performance costs.

### Performance Costs

This section outlines the main takeaways from the performance costs [measurements](https://docs.google.com/document/d/1ifY4dzNCstiTklYEBWMbyz5TeGoXzalYT7zEr5iWJ8Q/edit).

#### Pod Overhead

The overhead of starting a `Pod` is ~4s, as measured in our experiments with very simple `Pods`. While a 4s `Pod` overhead seems like a small fraction of the execution time of a typical CI/CD `Pipeline`, these performance costs add up. The `Pod` overhead increases linearly with an increasing number of sequential `Tasks`, for example a `Pipeline` with 5 sequential `Tasks` would have a `Pod` overhead of ~20s.

#### Workspace Overhead

As measured in out experiments, while the overhead of mounting an `EmptyDir Volume` is negligible, the overhead of mounting a `Persistent Volume` is ~10s for the first `Task` that uses it then becomes negligible for future `Tasks` that use it as long as they are on the same node.

In the case where `Pods` are scheduled to different nodes in a multi-node cluster, there’s an additional overhead of ~10s for node reattachment besides the ~10s of mounting a `Persistent Volume`. Users can schedule `Pods` to different nodes by disabling `Affinity Assistant`.

### Use Cases

1. As an Application Developer, I want to fetch the application source code from the repository and build a container image. 
2. As a Quality Assurance Engineer, I want to clone source code and run some end-to-end tests.
3. As a Data Engineer, I want to download blobs of user data from a GCS bucket, process the data and upload the results to another GCS bucket.
4. As an MLOps Engineer, I want to download a dataset from a GCS bucket to train an ML model.
5. As a DevOps Engineer, I want to optimize the performance of Tekton workloads by reducing the execution time and resource utilization.
6. As an Application Developer or Platform Engineer, I want to take parts of my pipelines and make them reusable by other teams.

### Related Work

#### Prior Art

##### 1.TaskGroup Custom Task

OpenShift has an experimental feature [TaskGroup](https://github.com/openshift-pipelines/tekton-task-group/tree/f43d027f4d5928e34d099b98870b17dbbffde65a) `Custom Task` that merges multiple `Tasks` into one `Task` that can be executed in one `Pod` with a shared context. With this option, users do not have to choose between reusability and performance. However, it is not easy to use because it depends on `Custom Tasks`.

|Task Group| Task|
|----------|-----|
|
```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskGroup
metadata:
  name: ci-tg
spec:
  workspaces:
    - name: shared-data
  params:
    - name: repo-url
    - name: revision
    - name: image-url
    - name: dockerfile
  results:
    - name: commit
    - name: digest
  steps:
  - uses:
      taskRef:
        name: git-clone
      parambindings:
      - name: url
        param: repo-url
      workspacebindings:
      - name: output
        param: shared-data
  - uses:
      taskRef:
        name: kaniko
      parambindings:
      - name: url
        param: image-url
      workspacebindings:
      - name: source
        param: shared-data
```
|
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: ci-tg
spec:
  workspaces:
    - name: shared-data
  params:
    - name: repo-url
    - name: revision
    - name: image-url
    - name: dockerfile
  results:
    - name: commit
    - name: digest
  steps:
  - name: git-clone-clone
    image: gcr.io/tekton-releases/git-init
    script: ...
  - name: kaniko-build-and-push
    workingDir: $(workspaces.source.path)
    image: gcr.io/kaniko-project/executor
    args: ...
``` 
|
