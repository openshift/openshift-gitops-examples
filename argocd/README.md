# ArgoCD GitOps Examples

<!-- TOC depthTo:3 -->

- [What is ArgoCD](#What-is-ArgoCD)
- [Prerequisites](#Prerequisites)
- [Installing ArgoCD on Openshift 4](#Installing-ArgoCD-on-OpenShift-4)
- [Configuring OpenShift 4](#Configuring-OpenShift-4)
    - [General Guidelines](#General-Guidelines)
    - [Examples](#Examples)
        - [Identity Provider](#Identity-Provider)
        - [Builds](#Builds)
        - [Registries](#Registries)
        - [Console](#Console)
        - [Scheduler Policy](#Scheduler-Policy)
        - [Machine Sets](#Machine-Sets)
        - [Operator Hub Operator](#Operator-Hub-Operator)
- [Multi-cluster Management](#Multi-cluster-Management)
    - [Deploy Configuration to Multiple Clusters](#Deploy-Configuration-to-Multiple-Clusters)
    - [Customizing Configuration By Cluster](#Customizing-Configuration-By-Cluster)

<!-- /TOC -->

# What is ArgoCD

ArgoCD is a declarative continuous delivery tool that leverages GitOps to maintain cluster resources. ArgoCD is implemented as a controller which is continuously monitoring application definitions and configurations defined in a Git repository and compares the desired state of those configurations with their live state on the cluster. Configurations which deviate from their desired state in the Git repository are classified as `OutOfSync`. ArgoCD reports these differences and allows administrators to automatically or manually resync configurations to the desired state.

# Prerequisites

The examples contained in this section require,

* the [oc](https://access.redhat.com/downloads/content/290) OpenShift client command-line tool
* a [kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) file for an existing OpenShift cluster (default location is `~/.kube/config`)
* the [argocd](https://github.com/argoproj/argo-cd/releases/latest) command-line tool

## Installing ArgoCD on OpenShift 4

These manual steps will hopefully be replaced by an ArgoCD operator on OperatorHub in the near future.

```bash
oc new-project argocd
oc apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
oc create route passthrough --service=argocd-server

# but this does not seem to work for console logins...
#oc apply -n argocd -f argocd.yaml
#oc create route edge --service=argocd-server

# Get the argoCD 'admin' password:
ARGO_ADMIN_PASS=`kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2`

# Login:
ARGO_ROUTE=`oc get route argocd-server -n argocd -o jsonpath='{.spec.host}'`
argocd login $ARGO_ROUTE:443 --username admin --password $ARGO_ADMIN_PASS --insecure

# Change the ArgoCD password:
argocd account update-password
```

NOTE: ArgoCD does not have any local users other than the built-in `admin` user. By default, only the `admin` user may interract with ArgoCD and its apps. Additional users can manage ArgoCD via SSO if configured. See the [ArgoCD Operator Manual](https://argoproj.github.io/argo-cd/operator-manual/sso/).

# Configuring OpenShift 4

## General Guidelines

 1. ArgoCD "Applications" (despite the name) can be used to deliver global custom resources such as those which configure OpenShift v4 clusters.
 1. When creating an application you will be required to provide a namespace. In the case of an application delivering global custom resources this doesn't make a lot of sense, but you can provide the name of any namespace to get past this issue.
 1. By default Argo will look to prune resources, should you ever delete your application that delivered them. In the case of OpenShift v4 global configuration custom resources, these often are blocked from being deleted, which can cause Argo to become stuck. If however in your configuration git repository you add the `argocd.argoproj.io/sync-options: Prune=false` annotation to your custom resources, this problem can be avoided. If you do run into this problem, you will need to manually "kubectl edit" the Argo Application and remove the finalizer which blocks until resources are pruned.

## Examples

The following section demonstrates the use of ArgoCD to deliver some of the available [OpenShift v4 Cluster Customizations](https://docs.openshift.com/container-platform/4.1/installing/install_config/customizations.html).

### Identity Provider

The [identity-providers](./identity-providers) directory contains an example for deploying an HTPasswd OAuth provider, and the associated secret. Deploying this as an ArgoCD application should allow you to login to your cluster as *user1 / MyPassword!*. For information on how this secret was created, see the [OpenShift 4 Documentation](https://docs.openshift.com/container-platform/4.1/authentication/identity_providers/configuring-htpasswd-identity-provider.html#configuring-htpasswd-identity-provider).

```bash
argocd app create htpasswd-oauth --repo https://github.com/openshift/openshift-gitops-examples.git --path=argocd/identity-providers --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync htpasswd-oauth
```

This example includes both a global OAuth config resource, and a namespaced secret.

WARNING: The openshift-oauth operator copies your specified secrets to the openshift-authentication, including their labels. One of these labels in added by ArgoCD to indicate the secret is owned by the htpasswd-oauth application. When this is copied, it causes ArgoCD to now see the copied secret as a resource it doesn't know about, is owned by this app, thus should be pruned. You can disable pruning with the normal annotation but will still see this secret as out of sync in the UI.

### Builds

The [builds](./builds) directory contains an example global Build configuration.

```bash
argocd app create builds-config --repo https://github.com/openshift/openshift-gitops-examples.git --path=argocd/builds/base --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync builds-config
```

### Registries

The [image](./image) directory contains an example global Image configuration which sets `allowedRegistriesForImport`, limiting the container image registries from which normal users may import images to only include `quay.io`.

```bash
argocd app create image-config --repo https://github.com/openshift/openshift-gitops-examples.git --path=argocd/image --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync image-config
```

### Console

The [console](./console) directory contains a simple configuration for the OpenShift console which simply changes the logout behavior to redirect to Google.

```bash
argocd app create console-config --repo https://github.com/openshift/openshift-gitops-examples.git --path=argocd/console --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync console-config
```

TODO: The --dest-namespace here is odd as this example contains only a global resource.


### Scheduler Policy

The [scheduler](./scheduler) directory contains an example scheduler policy configmap which can be deployed to override the default scheduler policy. For information regarding scheduler predicates, see the [OpenShift 4 Documentation](https://docs.openshift.com/container-platform/4.1/nodes/scheduling/nodes-scheduler-default.html#nodes-scheduler-default-predicates_nodes-scheduler-default).

```bash
argocd app create scheduler-policy --repo https://github.com/openshift/openshift-gitops-examples.git --path=argocd/scheduler --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-config
argocd app sync scheduler-policy
```

### Machine Sets

The [machine-sets](./machine-sets) directory contains an example `MachineSet` being deployed as an application via ArgoCD:

```bash
argocd app create machineset --repo https://github.com/openshift/openshift-gitops-examples.git --path=argocd/machine-sets --dest-server=https://kubernetes.default.svc --dest-namespace=openshift-machine-api
argocd app sync machineset
```

However there is a problem here, if you [view the yaml](./machine-sets/machinesets.yaml) you will see the cluster's generated InfraID referenced multiple times. This value is generated by the OpenShift installer and used in the naming of many cloud objects. Committing cluster config will be problematic as this value is not known before install, and not consistent across clusters.

A standard OpenShift 4 cluster with 3 compute nodes in us-east-1 comes with 6 MachineSets, one per AZ (in my account), with only three of them scaled to 1 replicas. Each MachineSet references the generated InfraID roughly 9 times:

 - MachineSet Name
 - Selector
 - IAM Instance Profile
 - Security Group Name
 - Subnet
 - AWS Tags

TODO: Should we recommend against using MachineSets with gitops and Argo? Or is there a templating solution we should explore? In this case the value we want to template is a fact about the individual cluster it's being deployed to.

### Operator Hub Operator

Deploy an operator from [Operator Hub](https://operatorhub.io/) by creating `OperatorGroup` and `Subscription` objects. In this example we will deploy the [grafana operator](https://operatorhub.io/operator/grafana-operator).

```
argocd app create grafana-operator --repo https://github.com/openshift/openshift-gitops-examples.git --path=argocd/grafana-operator --dest-server=https://kubernetes.default.svc --dest-namespace=default
argocd app sync grafana-operator
```


# Multi-cluster Management

In this example we will manage the build configuration of two OpenShift 4.x clusters, a pre-production (context: `pre`) cluster and a production (context: `pro`) cluster.

The example build configuration we will deploy contains customizations to be made per cluster environment.

## Deploy Configuration to Multiple Clusters

Ensure we have access to both clusters via kubeconfig context,

```bash
$ oc --context pre get nodes
NAME                           STATUS    ROLES     AGE       VERSION
ip-10-0-133-97.ec2.internal    Ready     master    5h        v1.14.6+7e13ab9a7
ip-10-0-136-91.ec2.internal    Ready     worker    5h        v1.14.6+7e13ab9a7
ip-10-0-144-237.ec2.internal   Ready     worker    5h        v1.14.6+7e13ab9a7
ip-10-0-147-216.ec2.internal   Ready     master    5h        v1.14.6+7e13ab9a7
ip-10-0-165-161.ec2.internal   Ready     master    5h        v1.14.6+7e13ab9a7
ip-10-0-169-135.ec2.internal   Ready     worker    5h        v1.14.6+7e13ab9a7
```

```bash
$ oc --context pro get nodes
NAME                           STATUS    ROLES     AGE       VERSION
ip-10-0-133-100.ec2.internal   Ready     master    5h        v1.14.6+7e13ab9a7
ip-10-0-138-244.ec2.internal   Ready     worker    5h        v1.14.6+7e13ab9a7
ip-10-0-146-118.ec2.internal   Ready     master    5h        v1.14.6+7e13ab9a7
ip-10-0-151-40.ec2.internal    Ready     worker    5h        v1.14.6+7e13ab9a7
ip-10-0-165-83.ec2.internal    Ready     worker    5h        v1.14.6+7e13ab9a7
ip-10-0-175-20.ec2.internal    Ready     master    5h        v1.14.6+7e13ab9a7
```

NOTE: Setting up multiple contexts with separate kubeconfigs can be achieved by [merging kubeconfigs](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).

In order to merge several kubeconfigs, ensure that each kubeconfig you wish to merge is configured with a user unique to the particular kubeconfig. For example, if each kubeconfig you wish to merge contains an `admin` user then that user would need to be changed to something unique to the cluster identified by the kubeconfig such as `admin1`. Simply update the user string in the kubeconfig.

For this example, we will have two kubeconfig files `cluster1.kubeconfig` and `cluster2.kubeconfig` that will be merged into `merged-config.kubeconfig`.

```bash
export KUBECONFIG="merged-config.kubeconfig:cluster1.kubeconfig:cluster2.kubeconfig"

$ oc config get-contexts 
CURRENT   NAME    CLUSTER    AUTHINFO    NAMESPACE
          admin1  cluster1   admin1
          admin2  cluster2   admin2

$ oc config set-context pre --cluster=cluster1 --user=admin1
Context "pre" created.

$ oc config set-context pro --cluster=cluster2 --user=admin2
Context "pro" created.
```

Next, ensure that each cluster has been registered with ArgoCD. Clusters are added to ArgoCD by specifying the context,

```bash
$ argocd cluster add
ERRO[0000] Choose a context name from:
CURRENT  NAME    CLUSTER    SERVER
         admin1  cluster1   https://api.cluster1.new-installer.openshift.com:6443
         admin2  cluster2   https://api.cluster2.new-installer.openshift.com:6443
*        pre     cluster1   https://api.cluster1.new-installer.openshift.com:6443
         pro     cluster2   https://api.cluster2.new-installer.openshift.com:6443

$ argocd cluster add pre
INFO[0000] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0000] ClusterRole "argocd-manager-role" created
INFO[0000] ClusterRoleBinding "argocd-manager-role-binding" created, bound "argocd-manager" to "argocd-manager-role"
Cluster 'pre' added

$ argocd cluster add pro
INFO[0000] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0000] ClusterRole "argocd-manager-role" created
INFO[0000] ClusterRoleBinding "argocd-manager-role-binding" created, bound "argocd-manager" to "argocd-manager-role"
Cluster 'pro' added

$ argocd cluster list
SERVER                                                  NAME  STATUS      MESSAGE
https://kubernetes.default.svc                                Successful
https://api.cluster2.new-installer.openshift.com:6443   pro   Successful
https://api.cluster1.new-installer.openshift.com:6443   pre   Successful
```

Add our build configuration repository to ArgoCD. The build configuration repository has a `pre` and `pro` kustomize overlay which will override the build `imageLabels` by cluster but we will start by deploying the base build configuration.

```bash
$ argocd repo add https://github.com/openshift/openshift-gitops-examples.git
```

Deploy custom OpenShift build configuration to pre-production and production clusters,

```bash
$ argocd app create --project default \
                   --name pre-builds \
                   --repo https://github.com/openshift/openshift-gitops-examples.git \
                   --path argocd/builds/base \
                   --dest-server https://api.cluster1.new-installer.openshift.com:6443 \
                   --dest-namespace=openshift-config \
                   --revision master

$ argocd app create --project default \
                   --name pro-builds \
                   --repo https://github.com/openshift/openshift-gitops-examples.git \
                   --path argocd/builds/base \
                   --dest-server https://api.cluster2.new-installer.openshift.com:6443 \
                   --dest-namespace=openshift-config \
                   --revision master
```

Sync configuration to both clusters as we have not defined an ArgoCD sync policy for the apps and must sync configurations manually.

```bash
$ argocd app sync pre-builds
$ argocd app sync pro-builds
```

Ensure both configurations have been successfully synced,

```bash
$ argocd app list
NAME        CLUSTER                                                 NAMESPACE         PROJECT  STATUS  HEALTH
pre-builds  https://api.cluster1.new-installer.openshift.com:6443   openshift-config  default  Synced  Healthy
pro-builds  https://api.cluster2.new-installer.openshift.com:6443   openshift-config  default  Synced  Healthy
```

Grab the modified build configuration from each cluster and ensure that it has been updated,

```bash
$ oc --context pre get build.config.openshift.io/cluster -o yaml -n openshift-config

$ oc --context pro get build.config.openshift.io/cluster -o yaml -n openshift-config
```

## Customizing Configuration By Cluster

In this example, we will modify our build configuration based on which cluster we are deploying to. ArgoCD leverages [kustomize](https://kustomize.io/) to manage configuration overrides across environments. In the `pre` and `pro` [overlay directories](https://github.com/dgoodwin/openshift4-gitops/tree/master/builds/overlays) of our git repository there are `kustomization` files which include patches to apply to the base configuration. We will specify the `overlays` directory containing our kustomizations as the application path instead of the `base` directory builds configuration directory.

Deploy kustomized build configuration to pre-production and production clusters,

```bash
$ argocd app create --project default \
                    --name pre-kustomize-builds \
                    --repo https://github.com/openshift/openshift-gitops-examples.git \
                    --path argocd/builds/overlays/pre \
                    --dest-server https://api.cluster1.new-installer.openshift.com:6443 \
                    --dest-namespace openshift-config \
                    --revision master \
                    --sync-policy automated

$ argocd app create --project default \
                    --name pro-kustomize-builds \
                    --repo https://github.com/openshift/openshift-gitops-examples.git \
                    --path argocd/builds/overlays/pro \
                    --dest-server https://api.cluster2.new-installer.openshift.com:6443 \
                    --dest-namespace openshift-config \
                    --revision master \
                    --sync-policy automated
```

Ensure that configuration applications have been synced successfully,

```bash
$ argocd app get pre-kustomize-builds
Name:               pre-kustomize-builds
Project:            default
Server:             https://api.cluster1.new-installer.openshift.com:6443
Namespace:          openshift-config
URL:                https://argocd-server-argocd.apps.cluster1.new-installer.openshift.com/applications/pre-kustomize-builds
Repo:               https://github.com/openshift/openshift-gitops-examples.git
Target:             pre
Path:               argocd/builds/overlays/pre
Sync Policy:        Automated
Sync Status:        Synced to master (884a6db)
Health Status:      Healthy

GROUP                KIND   NAMESPACE         NAME     STATUS   HEALTH   HOOK  MESSAGE
config.openshift.io  Build  openshift-config  cluster  Running  Synced         build.config.openshift.io/cluster configured
config.openshift.io  Build                    cluster  Synced   Unknown
```

```bash
$ argocd app get pro-kustomize-builds
Name:               pro-kustomize-builds
Project:            default
Server:             https://api.cluster2.new-installer.openshift.com:6443
Namespace:          openshift-config
URL:                https://argocd-server-argocd.apps.cluster2.new-installer.openshift.com/applications/pro-kustomize-builds
Repo:               https://github.com/openshift/openshift-gitops-examples.git
Target:             pro
Path:               argocd/builds/overlays/pro
Sync Policy:        Automated
Sync Status:        Synced to master (884a6db)
Health Status:      Healthy

GROUP                KIND   NAMESPACE         NAME     STATUS   HEALTH   HOOK  MESSAGE
config.openshift.io  Build  openshift-config  cluster  Running  Synced         build.config.openshift.io/cluster unchanged
config.openshift.io  Build                    cluster  Synced   Unknown
```

Grab the `imageLabels` which have been modified per environment using kustomize,

```bash
$ oc --context pre get build.config.openshift.io/cluster -n openshift-config -o jsonpath='{.spec.buildDefaults.imageLabels}'
[map[value:true name:preprodbuild]]

$ oc --context pro get build.config.openshift.io/cluster -n openshift-config -o jsonpath='{.spec.buildDefaults.imageLabels}'
[map[value:true name:prodbuild]]
```
