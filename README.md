# fun-with-sveltos

1. I already had a Management Cluster with HMC deployed on it, and a Target cluster named `aws-dev` deployed using the `aws-standalone-cp` template.

2. I installed Sveltos on it using Local Agent Mode as described in https://projectsveltos.github.io/sveltos/getting_started/install/install/.
```sh
➜  ~ ksveltos get deploy
Warning: short name "deploy" could also match lower priority resource deployments.hmc.mirantis.com
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
access-manager          1/1     1            1           21m
addon-controller        1/1     1            1           21m
classifier-manager      1/1     1            1           21m
conversion-webhook      1/1     1            1           21m
event-manager           1/1     1            1           21m
hc-manager              1/1     1            1           21m
sc-manager              1/1     1            1           21m
shard-controller        1/1     1            1           21m
sveltos-agent-manager   1/1     1            1           18m
```

3. After installation, I can see the following classifier:
```sh
➜  ~ ksveltos get classifier                           
NAME                 AGE
default-classifier   11m
```
If I get its status, I can see that it already identified that I have an `aws-dev` cluster deployed using CAPI and it also knows that it has been successfully provisioned.
```sh
➜  ~ ksveltos get classifier default-classifier -o yaml
apiVersion: lib.projectsveltos.io/v1beta1
kind: Classifier
metadata:
  annotations:
    projectsveltos.io/deployed-by-sveltos: "true"
  creationTimestamp: "2024-08-29T23:45:54Z"
  finalizers:
  - classifierfinalizer.projectsveltos.io
  generation: 1
  name: default-classifier
  resourceVersion: "165605"
  uid: 91bae8f1-9c71-4df6-9521-3e14979f49c5
spec:
  classifierLabels:
  - key: sveltos-agent
    value: present
status:
  clusterInfo:
  - cluster:
      apiVersion: cluster.x-k8s.io/v1beta1
      kind: Cluster
      name: aws-dev
      namespace: hmc-system
    hash: SbxtIl0OPjIuqduBHcCikmFE1xWRvTRtLYiiVMYlqPA=
    status: Provisioned
  - cluster:
      apiVersion: lib.projectsveltos.io/v1beta1
      kind: SveltosCluster
      name: mgmt
      namespace: mgmt
    hash: SbxtIl0OPjIuqduBHcCikmFE1xWRvTRtLYiiVMYlqPA=
    status: Provisioned
  machingClusterStatuses:
  - clusterRef:
      apiVersion: cluster.x-k8s.io/v1beta1
      kind: Cluster
      name: aws-dev
      namespace: hmc-system
    managedLabels:
    - sveltos-agent
  - clusterRef:
      apiVersion: lib.projectsveltos.io/v1beta1
      kind: SveltosCluster
      name: mgmt
      namespace: mgmt
    managedLabels:
    - sveltos-agent
```
All of this so far is most likely done by the `classifier-manager` when we applied the [default-classifier](https://raw.githubusercontent.com/projectsveltos/sveltos/main/manifest/default-classifier.yaml) as part of Sveltos installation.

4. Now if we look at the kube object for our `aws-dev` cluster, we will see the `sveltos-agent: present` label on it:
```sh
➜  ~ khmc get cluster aws-dev -o yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  annotations:
    meta.helm.sh/release-name: aws-dev
    meta.helm.sh/release-namespace: hmc-system
  creationTimestamp: "2024-08-27T12:38:06Z"
  finalizers:
  - cluster.cluster.x-k8s.io
  generation: 2
  labels:
    app.kubernetes.io/managed-by: Helm
    helm.toolkit.fluxcd.io/name: aws-dev
    helm.toolkit.fluxcd.io/namespace: hmc-system
    sveltos-agent: present
  name: aws-dev
  namespace: hmc-system
  resourceVersion: "165597"
  uid: b5495852-c4bd-4dba-a2cb-375aaa764ece
. . . . . . . . . . . .
```
I can also see 3 `classifierreports` present in the Management Cluster in different namespaces:
```sh
➜  ~ kubectl get classifierreports.lib.projectsveltos.io -A
NAMESPACE        NAME                                AGE
hmc-system       capi--default-classifier--aws-dev   96m
mgmt             sveltos--default-classifier--mgmt   96m
projectsveltos   default-classifier                  96m
```
Also I can see that the `sveltos-agent: present` laebl has been applied to our `aws-dev` target cluster created through CAPI:
```sh
➜  ~ khmc get cluster aws-dev -o yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  annotations:
    meta.helm.sh/release-name: aws-dev
    meta.helm.sh/release-namespace: hmc-system
  creationTimestamp: "2024-08-27T12:38:06Z"
  finalizers:
  - cluster.cluster.x-k8s.io
  generation: 2
  labels:
    app.kubernetes.io/managed-by: Helm
    helm.toolkit.fluxcd.io/name: aws-dev
    helm.toolkit.fluxcd.io/namespace: hmc-system
    sveltos-agent: present
  name: aws-dev
  namespace: hmc-system
. . . . . . . . . . . . .
```
How was Sveltos able to do all this? I am not sure, but I suspect something like the following happened:
  * a) Some component, most likely `sveltos-agent-manager` gathered information about the cluster it is running on (management cluster) and looked for CAPI clusters by looking for `cluster.x-k8s.io` type objects.
  * b) Some component (which one?) then installs the agent-manager into the target cluster adds the label `sveltos-agent: present` to the corresponding `aws-dev` Cluster object.
  * c) Some when we create the [default-classifier](https://raw.githubusercontent.com/projectsveltos/sveltos/main/manifest/default-classifier.yaml), it results in 3 `ClassifierReport` objects, each of which is [created by some component](https://github.com/projectsveltos/classifier/blob/662c8015887367bbca32604624c353277a632412/controllers/classifier_controller.go#L461-L467) somehow:
    ```sh
        ➜  ~ kubectl get classifierreports.lib.projectsveltos.io -A                                    
        NAMESPACE        NAME                                AGE
        hmc-system       capi--default-classifier--aws-dev   134m
        mgmt             sveltos--default-classifier--mgmt   134m
        projectsveltos   default-classifier                  135m
    ```
  * d) I think some component (probably agent-manager) evaluates these classifiers, performs some actions and generates a report.

5. In any case, I don't know how Sveltos does it but it seems auto-magical and I can see the agent-manager running on the target cluster:
```sh
➜  ~ kubectl get node
NAME             STATUS   ROLES           AGE     VERSION
ip-10-0-11-120   Ready    <none>          2d12h   v1.30.2+k0s
ip-10-0-7-183    Ready    control-plane   2d12h   v1.30.2+k0s
➜  ~ 
➜  ~ kubectl get ns
NAME              STATUS   AGE
cert-manager      Active   2d12h
default           Active   2d12h
k0s-autopilot     Active   2d12h
kube-node-lease   Active   2d12h
kube-public       Active   2d12h
kube-system       Active   2d12h
nginx-ingress     Active   2d12h
projectsveltos    Active   139m
➜  ~ 
➜  ~ kubectl -n projectsveltos get deploy
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
sveltos-agent-manager   1/1     1            1           139m

```