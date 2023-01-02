# GitLab Runner Kubernetes Executor with Google Kubernetes Engine Autopilot

This document details deployment of the [GitLab Runner](https://docs.gitlab.com/runner/) [Kubernetes Executor](https://docs.gitlab.com/runner/executors/kubernetes.html) [Helm Chart](https://gitlab.com/gitlab-org/charts/gitlab-runner) on [GKE Autopilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview).

While most elements work without modification, some Autopilot-specific changes are necessary.

## Changes to default [`values.yaml`](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml)

### Required changes

Below are the minimum number of changes to the default `values.yaml` to run in GKE Autopilot.

* Set `gitlabUrl` to URL of instance ([~L50](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L51)):

    `gitlabUrl: <URL to base GitLab install>`

* Set `runnerRegistrationToken` for project/group/subgroup ([~L55](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L57)):

    `runnerRegistrationToken: "<runner registration token>"`

* Enable rbac ([~L140](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L139)):

       rbac:
         create: true

* Define a node selector to ensure executor lives in a [separate workload](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-provisioning#workload_separation) to avoid preempting of the runner pod by jobs or cluster services ([~L620](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L617)):

       nodeSelector:
           group: autopilot-executor

* Set the corresponding toleration ([~L625](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L625)):

       tolerations:
         - key: group
           operator: Equal
           value: autopilot-executor
           effect: NoSchedule

### Recommended changes

* Set runner tag ([~L360](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L358)):

        tags: <runner tag>

* Diable running untagged ([~L375](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L374)):

        runUntagged: false

* Increase concurrency ([~L95](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L93)). Each 100 jobs consume approximately 0.25 CPU and 0.25 GB RAM.

        concurrent: 400

* Set higher runner pod resources ([~L600](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L601)):

        resources:
          cpu: 1
          memory: 1Gi

* Provide additional updates to job pods config, run all job pods as [spot VMs](https://cloud.google.com/kubernetes-engine/docs/concepts/spot-vms) ([~L315](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/blob/main/values.yaml#L317)):

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
