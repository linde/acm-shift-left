

* Anthos Config Management "Shift Left" Exploration

This is an example minial repo to explore the topic of guardrails as it relates
to enviroment configuration and how one might might implement at different
points in the config lifecycle.  The analogy is that in application development,
one tries to "shift left" testing so that it occurs as early in the cycle and as
far away from production as possible. Anthos Configuration Management (ACM)
provides tools to accomplish this for policy and configuration across clusters
and across clouds and on-prem.

* Problem Statement

Imagine Ida works in a company that has adopted controls such that all
namespaces in their platform have a label to idenify its cost centers.

* Getting Going

Let's get ACM installed and managing the namespace config on the
clusters. Follow these steps to [install and configure
ACM](https://cloud.google.com/anthos-config-management/docs/how-to/installing).

```bash
$ kubectl apply -f config-management-operator.yaml
$ kubectl create clusterrolebinding $(whoami)-cluster-admin-binding --clusterrole=cluster-admin --user=$(whoami)@google.co
m # for GKE
$ kubectl create secret generic git-creds -n=config-management-system --from-file=ssh=$HOME/.ssh/id_rsa.nomos
```

Then `kubectl apply -f [config-management.yaml](config-management.yaml)` and it
will take a few minutes to compile the tempaltes but eventually the cluster will
sync:





* Verify Admission Control

```bash
$ kubectl create namespace non-conforming
## TODO see error
```

* Check via GitOps and Shift Left pre-commit checks

```bash
$ docker run -it --rm -v $(pwd)/config-root:/workspace \ 
    gcr.io/kpt-dev/kpt cfg cat --wrap-kind=ResourceList --wrap-version=v1  /workspace | 
    docker run  -i gcr.io/kpt-functions/gatekeeper-validate:latest

```

* Leverage CI to do this for every pull request (PR)

```bash
# some stuff with cloudbuild
```






