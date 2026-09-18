# OpenShift Logging with LokiStack and Splunk

Kustomize configuration for Red Hat OpenShift Logging 6.x on OpenShift 4.20.
It deploys:

- **LokiStack** for in-cluster log storage, backed by S3-compatible object storage
- **ClusterLogForwarder**, which sends application, infrastructure and audit
  logs to both LokiStack and Splunk HTTP Event Collector (HEC)
- **UIPlugin**, which adds the **Observe → Logs** view to the OpenShift web console
- The collector service account and its cluster role bindings

The layout is a shared `base` plus one overlay per environment, so the same
configuration can later be managed by a GitOps tool such as Argo CD.

```
base/                 Shared resources (LokiStack size 1x.small)
test/                 Test overlay (LokiStack size 1x.demo)
  cluster.env.example   Splunk URL and StorageClass template
  patches/              Overlay patches
  secrets/              Credential templates
prod/                 Production overlay (placeholder)
```

## Prerequisites

### Operators

Install the following operators from OperatorHub before applying this
configuration. They are not managed by this repository.

| Operator | Install namespace | Validated version |
|---|---|---|
| Red Hat OpenShift Logging | `openshift-logging` | 6.6 |
| Loki Operator | `openshift-operators-redhat` | 6.6 |
| Cluster Observability Operator | `openshift-cluster-observability-operator` | 1.5 |

This configuration doesn't manage the `openshift-logging` namespace. The Logging
Operator is installed there, and removing this configuration must leave the
operator in place.

### Object storage

LokiStack requires an S3-compatible bucket, such as AWS S3, MinIO or OpenShift
Data Foundation. Prepare:

- A dedicated bucket for Loki. Leave versioning and object locking disabled,
  because Loki manages retention itself.
- Credentials limited to that bucket. A minimal policy:

  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:*"],
        "Resource": [
          "arn:aws:s3:::<bucket>",
          "arn:aws:s3:::<bucket>/*"
        ]
      }
    ]
  }
  ```

- An HTTPS endpoint whose certificate the cluster trusts. The Loki Operator
  doesn't support skipping certificate verification for object storage. If
  the certificate comes from a private CA, set `spec.storage.tls.caName` on the
  LokiStack to a ConfigMap containing that CA.

### Splunk

- HTTP Event Collector enabled, with SSL
- A HEC token for OpenShift, with a default index such as `main` or a dedicated
  index
- A HEC certificate trusted by the cluster. For a private CA, add a `tls.ca`
  reference to the Splunk output. Don't disable verification.

Check the endpoints from a workstation before deploying. Don't use `-k`:
certificate verification must succeed.

```bash
curl -sS https://<s3-endpoint>/minio/health/live -o /dev/null -w '%{http_code}\n'   # MinIO only
curl -sS https://<splunk-host>:8088/services/collector/health
curl -sS -H "Authorization: Splunk <token>" -X POST https://<splunk-host>:8088/services/collector/event
# expected: {"text":"No data","code":5}. The token is valid; nothing is indexed.
```

### StorageClass

LokiStack needs a StorageClass for its persistent volumes. Block storage is
recommended. List the available classes with `oc get storageclass`.

## Configuration

Environment-specific values and credentials live in local files that are
excluded from version control by `.gitignore`. Create them from the templates:

```bash
cd test
cp cluster.env.example            cluster.env
cp secrets/loki-s3.env.example    secrets/loki-s3.env
cp secrets/splunk-hec.env.example secrets/splunk-hec.env
```

| File | Key | Description |
|---|---|---|
| `cluster.env` | `splunkUrl` | Splunk HEC URL, e.g. `https://splunk.example.com:8088` |
| | `storageClassName` | StorageClass for LokiStack volumes |
| `secrets/loki-s3.env` | `access_key_id`, `access_key_secret` | Object storage credentials |
| | `bucketnames` | Bucket name |
| | `endpoint` | S3 endpoint URL |
| | `region` | Region (any value for MinIO) |
| | `forcepathstyle` | `true` for MinIO and most non-AWS S3 services |
| `secrets/splunk-hec.env` | `hecToken` | Splunk HEC token |

Values in `.env` files must not be quoted.

## Deployment

Log in to the target cluster with `oc`, then:

```bash
oc kustomize test/                   # preview the rendered manifests
oc apply -k test/ --dry-run=server   # validate against the cluster
oc apply -k test/                    # apply or update
```

Re-run `oc apply -k test/` after changing any value. Both operators watch their
Secrets and reconcile changes.

### Verification

```bash
oc get pods -n openshift-logging
oc get lokistack logging-loki -n openshift-logging \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason}{"\n"}{end}'
oc get clusterlogforwarder collector -n openshift-logging \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.message}{"\n"}{end}'
```

- **LokiStack:** the `Ready` condition becomes `True`, usually within a few minutes.
- **Collector:** there is one collector pod per node.
- **Console:** logs appear under **Observe → Logs**. Reload the console if the
  menu entry doesn't show up.
- **Splunk:** search the configured index, for example
  `index=main kubernetes.namespace_name=*`. The `log_type` field
  distinguishes `application`, `infrastructure` and `audit`.

## Removal

```bash
oc delete -k test/
```

The operators and the `openshift-logging` namespace are left in place. LokiStack
PersistentVolumeClaims are also kept. To delete stored logs:

```bash
oc delete pvc -n openshift-logging -l app.kubernetes.io/instance=logging-loki
```

Objects in the S3 bucket aren't deleted.
