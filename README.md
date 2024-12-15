#Setup LokiStack with Grafana for Application Logs on OCP


This project is an example of how to acheive the following:
1. Setup Openshift logging
2. Install a LokiStack instance for application logs
3. Forward application logs from the Openshift logging stack to Loki
4. Setup a Grafana instance for a dev team which has dashboards for the specific team's namespaces
5. Manage all in argocd with auto-sync

## Prerequisites

In this example, the devteam is 'rocko'.  Start by creating a namespace for the grafana for team rocko

```
oc new-project rocko-grafana
```

### Setup minio for S3

minio helm:

https://artifacthub.io/packages/helm/bitnami/minio

Create minio with the following commands:
```
oc new-project minio
helm install minio oci://registry-1.docker.io/bitnamicharts/minio -n rocko-grafana
```

Create a route for the minio console

```
oc apply -f minio-route.yaml
```

Retrieve the minio admin password:

```
oc get secret minio -o yaml|grep -i root-password|awk '{print $2}'|base64 -d
```

Open the route created above and login using user admin and the password from the last command's ouput

Create a bucket

Open UI:  https://mino-minio.apps.<yourClusterDomain>/login

![Alt text](screenshots/miniologin.jpg?raw=true "Minio Login")



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
