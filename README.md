# Setup LokiStack with Grafana for Application Logs on OCP


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

![Alt text](screenshots/miniologin.jpeg?raw=true "Minio Login")

Login with the user admin and password retrieved above

Create Bucket 'loki'

![Alt text](screenshots/miniobucket.jpeg?raw=true "Minio Create Bucket")

Create an access key to be used when setting up LokiStack

![Alt text](screenshots/minioaccesskey.jpeg?raw=true "Minio Create Access Key")

### Setup Grafana Operator

grafana operator helm:

https://artifacthub.io/packages/helm/bitnami/grafana-operator

```
oc new-project grafana-operator
helm install grafana-operator -f values-grafana.yaml oci://registry-1.docker.io/bitnamicharts/grafana-operator -n grafana-operator
```

Grafana references:

https://github.com/grafana/grafana-operator/tree/master/examples


## Deploy Loki Stack in Openshift Cluster Logging

Install the operator using the OperatorHub

![Alt text](screenshots/lokioperator.jpeg?raw=true "Loki Operator")


Create Secret for LokiStack to connect to the minio bucket

```
apiVersion: v1
kind: Secret
metadata:
  name: logging-loki-s3
  namespace: openshift-logging
stringData:
  access_key_id: <updateWithYourKey>
  access_key_secret: <updateWithYourSecret>
  bucketnames: loki
  endpoint: http://minio.minio.svc:9000
```

```
oc apply -f loki/ocp/secret.yaml
```

Deploy LokiStack

```
oc apply -f loki/ocp/lokistack.yaml
```

Deploy ClusterLogging

reference doc: https://docs.openshift.com/container-platform/4.14/observability/logging/log_storage/installing-log-storage.html

Install the ClusterLogging operator

Deploy clusterlogging instance:

```
oc apply -f loki/ocp/clusterlogging.yaml
```

You should now have logs in the OCP UI under Observe -> Logs

![Alt text](screenshots/lokiloginsui.jpeg?raw=true "Loki Logs in OCP UI")


Extract login details for Loki

Retreive ca cert and access cert
```
oc get secret lokistack-sample-gateway-client-http -o yaml -n openshift-logging|grep -i tls.crt|awk '{print $2}'|base64 -d
```
The first cert in the list will be the access cert and the second the ca cert.  You can check this by copying the certificates into 2 separate files,
and running `openssl x509 -in <certfile.crt> -noout -text` and looking for the CA: TRUE field.  The one with this field is the ca cert.

Retreive ca key
```
oc get secret lokistack-sample-gateway-client-http -o yaml -n openshift-logging|grep -i tls.key|awk '{print $2}'|base64 -d
```

Update the values in the grafana/datasource.yaml as follows:

```
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDatasource
metadata:
  name: loki-datasource
  namespace: rocko-grafana
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  datasource:
    name: loki
    type: loki
    access: proxy
    url: https://lokistack-sample-querier-http.openshift-logging.svc:3100
    isDefault: true
    secureJsonData:
      tlsCACert: |
        -----BEGIN CERTIFICATE-----
        < Your CA Cert that was extracted from the previous command >
        -----END CERTIFICATE-----
      tlsClientCert: |
        -----BEGIN CERTIFICATE-----
        < Your Accessd Cert that was extracted from the previous command >
        -----END CERTIFICATE-----
      tlsClientKey: |
        -----BEGIN RSA PRIVATE KEY-----
        < Your Key that was extracted from previous command >
        -----END RSA PRIVATE KEY-----
    jsonData:
      "tlsAuth": true
      "tlsAuthWithCACert": true
      httpHeaderName1: "X-Scope-OrgID"
    secureJsonData:
      "httpHeaderValue1": "application"
```

Install Clusterlogging stack with Loki

https://docs.openshift.com/container-platform/4.14/observability/logging/log_storage/installing-log-storage.html
