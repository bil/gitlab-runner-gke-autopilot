# GitLab Runner Kubernetes Executor with Google Kubernetes Engine Autopilot

This document details deployment of the [GitLab Runner](https://docs.gitlab.com/runner/) [Kubernetes Executor](https://docs.gitlab.com/runner/executors/kubernetes.html) [Helm Chart](https://gitlab.com/gitlab-org/charts/gitlab-runner) on [GKE Autopilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview).

While most elements work without modification, some Autopilot-specific changes are necessary.

## Changes to `values.yaml`

### Required changes

Below are the minimum number of changes to the default `values.yaml` to run in GKE Autopilot.

* `gitlabUrl: <URL to base GitLab install>`
* `runnerRegistrationToken: "<runner registration token>"
* Enable rbac:

       rbac:
         create: true

* Define a node selector to ensure executor lives in a [separate workload](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-provisioning#workload_separation) to avoid preempting of the runner pod by jobs or cluster services:

       nodeSelector:
           group: autopilot-executor

* Set the corresponding toleration:

       tolerations:
         - key: group
           operator: Equal
           value: autopilot-executor
           effect: NoSchedule

### Recommended changes

* Set runner tag:

        tags: <runner tag>

* Diable running untagged:

        runUntagged: false

* Increase concurrency. Each 100 jobs consume approximately 0.25 CPU and 0.25 GB RAM.

        concurrent: 400

* Set higher runner pod resources:

        resources:
          cpu: 1
          memory: 1Gi
j
* Provide additional updates to job pods config, run all job pods as [spot VMs](https://cloud.google.com/kubernetes-engine/docs/concepts/spot-vms):

        config: |
          [[runners]]
            [runners.kubernetes]
              namespace = "{{.Release.Namespace}}"
              image = "ubuntu:22.04"
              poll_timeout = 3600
              cpu_request = "500m"
              cpu_request_overwrite_max_allowed = 50
              memory_request = "256M"
              memory_request_overwrite_max_allowed = "256G"
              ephemeral_storage_request = "10G"
              ephemeral_storage_request_overwrite_max_allowed = "5T"
              helper_cpu_request = "500m"
              helper_cpu_request_overwrite_max_allowed = "5"
              helper_memory_request = "256M"
              helper_memory_request_overwrite_max_allowed = "2G"
              helper_ephemeral_storage_request = "5G"
              helper_ephemeral_storage_request_overwrite_max_allowed = "20G"
            [runners.kubernetes.node_selector]
              "cloud.google.com/gke-spot" = "true"
 
## Deployment

Deploy with [helm](https://helm.sh), recommend deploying from [Google Cloud Shell](https://cloud.google.com/shell) after [setting a default kubectl cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#default_cluster_kubectl):

    helm install gitlab-runner -f values.yaml gitlab/gitlab-runner

Subsequent updates to `values.yaml` configuration can be propogated with `helm upgrade`:

    helm upgrade gitlab-runner -f values.yaml gitlab/gitlab-runner

## Usage

Run [computational pipelines](https://en.wikipedia.org/wiki/Pipeline_(computing)) with [GitLab CI/CD](https://docs.gitlab.com/ee/ci) defined in a repository's [`.gitlab-ci.yml`](https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.html).

Resources can be specified on a per-job basis using the `KUBERNETES_` [container override variables](https://docs.gitlab.com/runner/executors/kubernetes.html#overwriting-container-resources).
