Install [app-operator](https://github.com/giantswarm/app-operator) on Management cluster

```
k apply -f app-crd.yaml
k apply -f appcatalog-crd.yaml
k apply -f appcatalogentries-crd.yaml
k apply -f app-operator.yaml
```

Add default and playground catalog

```
k apply -f default-catalog.yaml
k apply -f playground-catalog.yaml
```

Create namespace and add kubeconfig for CAPI cluster

```
k create ns cluster-01
k -n cluster-01 create secret generic cluster-01-kubeconfig --from-file=kubeConfig=./capi-quickstart.kubeconfig
```

Install chart-operator to capi cluster

```
k --kubeconfig=./capi-quickstart.kubeconfig \
  create priorityclass giantswarm-critical --value=1000000000
k -n cluster-01 apply -f chart-operator-app.yaml
```

Label the master node

```
k --kubeconfig=./capi-quickstart.kubeconfig \
  label node ip-10-0-186-81.eu-central-1.compute.internal kubernetes.io/role=master
```

Install a small app collection (metrics-server, cert-manager, ingress-nginx, prometheus-operator, and loki)

```
k -n cluster-01 apply -f app-collection.yaml
```
