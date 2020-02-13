
## Anthos Config Management Shift Left Exploration

This is an example minimal repo to explore the topic of guardrails as it relates
to enviroment configuration and how one might might implement controls at
different points in the config lifecycle.  The "shift left" analogy relates to 
application development where one tries to shift left testing time-wise so that it occurs
as early in the cycle and as far away from production as possible. Anthos Configuration
Management (ACM) provides tools to accomplish this for policy and configuration
across clusters and across clouds and on-prem.

## Problem Statement

Imagine Ida works in a company that has adopted controls such that all
namespaces in their platform need a label to identify their cost center for
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

Then [`kubectl apply -f config-management.yaml`](./config-management.yaml) and it will take a few minutes
to compile the tempaltes but eventually the cluster will sync:

```bash
$ nomos status 
Connecting to clusters...
Current   Context                 Status           Last Synced Token   Sync Branch
-------   -------                 ------           -----------------   -----------
*         kubernetes-admin@amex   SYNCED           f83841ea            master
```

## Verify Admission Control

Now let's take on enforcement of that accounting control for cost-center labels -- to do this, we will enlist the assitance of the [Anthos Config Management Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller). In our config, we have [enabled this controller](config-management.yaml#L8-L10) and configured a constraint through [k8srequiredlabels.yaml](./config-root/cluster/k8srequiredlabels.yaml) that customizes our policy for the specific label we require (ie "[cost-center](config-root/cluster/ns-should-have-cost-center.yaml#L13)"). 

To see this in action and make sure we're enforcing our accounting control, let's try to violate it:

```bash
$ kubectl create ns out-of-compliance-ns
Error from server ([denied by ns-cost-center] you must provide labels: {"cost-center"}): admission webhook "validation.gatekeeper.sh" denied the request: [denied by ns-cost-center] you must provide labels: {"cost-center"}
```

We can see this is the error message from the constraint in our cluster scoped
[k8srequiredlabels.yaml](./config-root/cluster/k8srequiredlabels.yaml) resource. Our controls are working! 


## Check via GitOps and Shift Left Pre-Commit checks

But depending on admission checks is like testing application code only deployed to prod. 
Let's try to shift this to the left.

### First, Linting

Let's try to put that Namespace into source control and have ACM actuate it instead of applying it directly. This gives us a chance to run some linting checks to make sure it can be applied.

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

> Note: ACM can handle [unstructured
repos](https://cloud.google.com/anthos-config-management/docs/how-to/unstructured-repo),
> but we're in its default mode which expects namespace resources in
> appropriate leaf level directories. ([learn more about it's
repo](https://cloud.google.com/anthos-config-management/docs/how-to/repo))

Move the violating config to the appropriate leaf directory and see that now
succeeds when vetted:

```bash
$ mkdir config-root/namespaces/vandelay-dev/
$ mv config-root/namespaces/vandelay-dev.yaml config-root/namespaces/vandelay-dev/namespace.yaml
$ nomos vet --path config-root/
$
```
`nomos vet` has no output when there are no issues, so we have no linting errors at this point.


### Next, Validation

Now, let's make sure our config doesn't just pass linting tests, but that it
complies with the accounting cost-center control too. 

To do this, we can use [`kpt`](https://github.com/GoogleContainerTools/kpt) to
manipulate the config and to run functions from its [KPT Functions
Catalog](https://googlecontainertools.github.io/kpt-functions-catalog/). 
In this case, we will use the [policy-controller-validate](http://gcr.io/config-management-release/policy-controller-validate)
function on the config to validate it against our [cost center
constraint](./config-root/cluster/ns-should-have-cost-center.yaml) above.

```bash
$ docker run -it --rm -v $(pwd)/config-root:/workspace/config-root \ 
    gcr.io/config-management-release/kpt cfg cat --wrap-kind=ResourceList  /workspace | 
    docker run  -i gcr.io/config-management-release/policy-controller-validate 
Error: Found 1 violations:

[1] you must provide labels: {"cost-center"}

name: "vandelay-dev"
```

Let's fix the label for the namespace (see this [line for an example](config-root/namespaces/vandelay-dev/namespace.yaml#L6)), then try again. If it passes through the
config without output to stderr, it worked!

```bash
$ docker run -it --rm -v $(pwd)/config-root:/workspace/config-root \ 
    gcr.io/config-management-release/kpt cfg cat --wrap-kind=ResourceList  /workspace | 
    docker run  -i gcr.io/config-management-release/policy-controller-validate 
apiVersion: v1
items:
- apiVersion: templates.gatekeeper.sh/v1beta1
  kind: ConstraintTemplate
  metadata:
...
```

## Leverage CI to do this for every pull request (PR)

Now that we have a useful validation flow, let's put that into a build step so it occurs on any 
pull request. To see this, take a look at the [cloudbuild.yaml](cloudbuild.yaml) and it should seem 
quite familiar -- this is simply just the CLI docker commands put into a trivial build step and trigger. 
To see an example run, consider [runs/442513791](../../runs/442513791).


## Conclusion

Wrapping this all up, we've seen that it is worthwhile to have controls for your configuration and even better
to shift left those checks. Not only does this help you find issues sooner, but it reinforces a defense in depth 
posture and helps teams establish guard rails for decentralized control, but with safeguards.






