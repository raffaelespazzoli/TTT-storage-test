# Storage, Pods and availability zone

This tutorial assuems you have provisioned a startend AWS 4.0 cluster, with 3 availability zones.

create a project for this tutorial

```shell
oc new-project storage-test1
```

## ReplicaSet and PVC

Create a PVC

```shell
oc apply -f ./pvc.yaml
```

examine the availability zone where the pv was created

```shell
oc get pv $(oc get pvc mypvc --no-headers | awk '{ print $3}') -o yaml | grep failure-domain.beta.kubernetes.io/zone
```

locate the `failure-domain.beta.kubernetes.io/zone` label

create the pod and verify that it has been scheduled in the same zone

```shell
oc adm policy add-scc-to-user anyuid -z default -n storage-test1
oc apply -f ./pod.yaml
POD_NAME=$(oc get pods --no-headers | awk '{print $1}')
oc get node $(oc get pod $POD_NAME -o wide --no-headers | awk '{ print $7}') -o yaml | grep failure-domain.beta.kubernetes.io/zone
```

Identify the `failure-domain.beta.kubernetes.io/zone` label and verify that corresponds to the pod's az.


## what happens when an entire az it not available.

identify the replicaset wihch is dedicated to the zone in which the pv is

```shell
ZONE=$(oc get pv $(oc get pvc mypvc --no-headers | awk '{ print $3}') -o yaml | grep failure-domain.beta.kubernetes.io/zone | awk '{print $2}')
MACHINE_SET=$(oc get machineset -n openshift-cluster-api --show-labels | grep $ZONE | awk '{print $1}')
```

scale the machine set to zero. This will cause the pods to be drained, then we should see that the pod does not get rescheduled

```shell
oc scale machineset $MACHINE_SET -n openshift-cluster-api --replicas=0
```

It takes a few minutes for the node to disappear.

```shell
oc get nodesfor i in $(oc get crd --no-headers | awk '{print $1}'); do echo $i $(oc get $i --no-headers --all-namespaces | awk '{print $1}')  ; done > allops.txt

```

when you see only two worker nodes verify that the pod is pending state

```shell
oc get pods
```


## clean up

```shell
oc scale machineset $MACHINE_SET -n openshift-cluster-api --replicas=1
oc delete project storage-test
```

## Bonus

This commands prints all the CRDs with an instance. This roughly correspnd to the all the operator configurations at install time

type 

```shell
for i in $(oc get crd --no-headers | awk '{print $1}'); do echo $i -n $(oc get $i --no-headers --all-namespaces | awk '{print $1 + " " + $2}') -o yaml  ; done > allops.txt
```
