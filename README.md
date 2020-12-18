# CAPA Walkthrough

This is mainly a shorter version of the [CAPI Quickstart](https://cluster-api.sigs.k8s.io/user/quick-start.html) plus some additional testing shenanigans and an excursion into installing apps with [app-operator](https://github.com/giantswarm/app-operator).

Export AWS CLI credentials

```
export AWS_REGION=eu-central-1
export AWS_ACCESS_KEY_ID=GETYOUROWN
export AWS_SECRET_ACCESS_KEY=GETYOUROWN
```

The clusterawsadm utility takes the credentials that you set as environment
variables and uses them to create a CloudFormation stack in your AWS account
with the correct IAM resources.

```
clusterawsadm bootstrap iam create-cloudformation-stack
```

Create the base64 encoded credentials using clusterawsadm.
This command uses your environment variables and encodes
them in a value to be stored in a Kubernetes Secret.

```
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)
```

Finally, initialize the management cluster (run this on a GS TC or kind or whatever K8s cluster you have handy)

```
clusterctl init --infrastructure aws
```

For SSH add your keypair named `default` to AWS see https://cluster-api-aws.sigs.k8s.io/topics/using-clusterawsadm-to-fulfill-prerequisites.html#ssh-key-pair

Create AWS cluster config

```
export AWS_REGION=eu-central-1
export AWS_SSH_KEY_NAME=default
export AWS_CONTROL_PLANE_MACHINE_TYPE=t3.large
export AWS_NODE_MACHINE_TYPE=t3.large
```

Generate CRs

```
clusterctl config cluster capi-quickstart \
  --kubernetes-version v1.19.5 \
  --control-plane-machine-count=1 \
  --worker-machine-count=3 \
  > capi-quickstart.yaml
```

Apply Cluster

```
kubectl apply -f capi-quickstart.yaml
```

Check Cluster

```
kubectl get cluster --all-namespaces
kubectl get kubeadmcontrolplane --all-namespaces
```

Get kubeconfig

```
clusterctl get kubeconfig capi-quickstart > capi-quickstart.kubeconfig
```

Apply Calico and check

```
kubectl --kubeconfig=./capi-quickstart.kubeconfig \
  apply -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
kubectl --kubeconfig=./capi-quickstart.kubeconfig get nodes
kubectl --kubeconfig=./capi-quickstart.kubeconfig get pods -A
```

CNCF conformance

```
export KUBECONFIG=~/Dev/CAPI/capi-quickstart.kubeconfig
sonobuoy run --mode=certified-conformance --wait
results=$(sonobuoy retrieve) && sonobuoy results $results
```

CIS conformance

```
sonobuoy run \
  --plugin https://raw.githubusercontent.com/vmware-tanzu/sonobuoy-plugins/master/cis-benchmarks/kube-bench-plugin.yaml \
  --plugin https://raw.githubusercontent.com/vmware-tanzu/sonobuoy-plugins/master/cis-benchmarks/kube-bench-master-plugin.yaml \
  --wait
outfile=$(sonobuoy retrieve) && sonobuoy results $outfile
```

For SSH see https://cluster-api-aws.sigs.k8s.io/topics/accessing-ec2-instances.html

For Installing apps see the [App Walkthrough](/app-walkthrough.md)

Cleanup

```
kubectl delete cluster capi-quickstart
```

Notes
- No PSP
- No Autoscaling
- No metrics server, basically bare bones and no addons, but that's on purpose
- sonobuoy got evicted, dug deeper and created upstream issue in CAPA to not use root disk for everything
- scaling from 3 to 4 workers, was simple edit of MachineDeployment replicas
- CIS conformance is quite high, some minor things missing (same as kubeadm)

#Upgrading from 1.18 to 1.19
- change version of CP first
- a new master is spawned next to the current one
- old master gets cordoned and drained once new one is Ready
- zero-downtime master upgrade
- same with machinedeploy
- adds a new node, waits for Ready, starts draining old node, and adds another node (not waiting for first drain to be done), every time a new one is Ready and old one starts getting drained
- smooth upgrade

#Switching the Machine types
- create new MachineTemplates with new settings
- change reference of CP or MP
- rolls through like a normal upgrade