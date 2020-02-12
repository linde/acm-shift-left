
## Anthos Config Management "Shift Left" Exploration

This is an example minial repo to explore the topic of guardrails as it relates
to enviroment configuration and how one might might implement controls at
different points in the config lifecycle.  The analogy is that in application
development, one tries to "shift left" testing so that it occurs as early in the
cycle and as far away from production as possible. Anthos Configuration
Management (ACM) provides tools to accomplish this for policy and configuration
across clusters and across clouds and on-prem.

## Problem Statement

Imagine Ida works in a company that has adopted controls such that all
namespaces in their platform need a label to identify its cost center for
accounting purposes.

## Getting Going

To get going, let's get ACM installed and managing the namespace config on the
clusters. Follow these steps to [install and configure
ACM](https://cloud.google.com/anthos-config-management/docs/how-to/installing).

```bash
$ kubectl apply -f config-management-operator.yaml
$ kubectl create clusterrolebinding $(whoami)-cluster-admin-binding --clusterrole=cluster-admin --user=$(whoami)@google.com # for GKE
$ kubectl create secret generic git-creds -n=config-management-system --from-file=ssh=$HOME/.ssh/id_rsa.nomos
```

Then `kubectl apply -f [config-management.yaml]` and it will take a few minutes
to compile the tempaltes but eventually the cluster will sync:

```bash
$ nomos status 
Connecting to clusters...
Current   Context                 Status           Last Synced Token   Sync Branch
-------   -------                 ------           -----------------   -----------
*         kubernetes-admin@amex   SYNCED           f83841ea            master
```

## Verify Admission Control

To make sure we're enforcing our accounting control, let's try to violate it:

```bash
$ kubectl create ns out-of-compliance-ns
Error from server ([denied by ns-cost-center] you must provide labels: {"cost-center"}): admission webhook "validation.gatekeeper.sh" denied the request: [denied by ns-cost-center] you must provide labels: {"cost-center"}
```

We can see this is the error message from the constraint in
[config-root/cluster/ns-should-have-cost-center.yaml]. Our controls are working!


## Check via GitOps and Shift Left Pre-Commit checks

### First, Linting

Let's try to put that Namespace into source control and have ACM actuate it
instead. Create a Namespace file and let's run some linting checks to make sure
it can be applied.

```bash
$ cat >config-root/namespaces/vandelay-dev.yaml <<END
apiVersion: v1
kind: Namespace
metadata:
  name: vandelay-dev
END
$ nomos vet --path config-root/
errors for Cluster "defaultcluster": 3 error(s)


[1] KNV1019: Namespaces MUST be declared in subdirectories of namespaces/. Create a subdirectory for Namespaces declared in:

source: namespaces/namespace.yaml
metadata.name: vandelay-dev
group:
version: v1
kind: Namespace

For more information, see https://cloud.google.com/anthos-config-management/docs/reference/errors#knv1019
```

That was helpful, we now know it was in the wrong directory! 

Note: ACM can handle [unstructured
repos](https://cloud.google.com/anthos-config-management/docs/how-to/unstructured-repo),
but in its default mode, which we're in, it expects namespace resources in
appropriate leaf level directories. ([learn more about it's
repo](https://cloud.google.com/anthos-config-management/docs/how-to/repo))

Move the violating config to the appropriate leaf director and see that now
succeeds when vetted:

```bash
$ mkdir config-root/namespaces/vandelay-dev/
$ mv config-root/namespaces/vandelay-dev.yaml config-root/namespaces/vandelay-dev/namespace.yaml
$ ~/proj/nomos/dist/nomos vet --path config-root/
$
```
`nomos vet` has no output when there are no issues, so we have no linting errors at this point.


### Next, Validation

Now, let's make sure our config does just pass linting tests, but that it
complies with Ida's cost-center policies too. 

To do this, we can use [`kpt`](https://github.com/GoogleContainerTools/kpt) to
manipulate the config and to run functions from its [KPT Functions
Catalog](https://googlecontainertools.github.io/kpt-functions-catalog/). In this
case, we will use the
[gcr.io/kpt-functions/gatekeeper-validate](http://gcr.io/kpt-functions/gatekeeper-validate)
function on the config to validate it against our [cost center
constraint](./config-root/cluster/ns-should-have-cost-center.yaml) above.

```bash
$ docker run -it --rm -v $(pwd)/config-root:/workspace/cluster \
    gcr.io/kpt-dev/kpt cfg cat --wrap-kind=ResourceList  /workspace | 
    docker run  -i gcr.io/kpt-functions/gatekeeper-validate:dev
Error: Found 1 violations:

[1] you must provide labels: {"cost-center"}

name: "vandelay-dev"
path: ?

```

Let's fix the label for the namespace, then try again. If it passes through the
config without output to stderr, it worked!

```bash
$ docker run -it --rm -v $(pwd)/config-root:/workspace/cluster  \
    gcr.io/kpt-dev/kpt cfg cat --wrap-kind=ResourceList  /workspace | 
    docker run  -i gcr.io/kpt-functions/gatekeeper-validate:dev 
apiVersion: v1
items:
- apiVersion: templates.gatekeeper.sh/v1beta1
  kind: ConstraintTemplate
  metadata:
...
```

## Leverage CI to do this for every pull request (PR)

```bash
# some stuff with cloudbuild
```






