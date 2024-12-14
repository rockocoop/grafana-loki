minio helm:

https://artifacthub.io/packages/helm/bitnami/minio

```
oc new-project minio
helm install minio oci://registry-1.docker.io/bitnamicharts/minio
```


grafana operator helm:

https://artifacthub.io/packages/helm/bitnami/grafana-operator

```
oc new-project grafana-operator
helm install grafana-operator -f values-grafana.yaml oci://registry-1.docker.io/bitnamicharts/grafana-operator
```

Grafana references:

https://github.com/grafana/grafana-operator/tree/master/examples


Extract login details for Loki

``` 
oc get secret lokistack-sample-gateway-client-http -o yaml
```



Install Clusterlogging stack with Loki

https://docs.openshift.com/container-platform/4.14/observability/logging/log_storage/installing-log-storage.html
