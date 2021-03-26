# postgres-in-k8s
POC using Zalando's [PostgreSQL K8s Operator](https://github.com/zalando/postgres-operator).


### Requirements

- A running Kubernetes Cluster.
    - Priviledged podSecurityPolicy installed.
- Helm v3.
- Access to an S3 service.


### Get Zalando's postgres-operator repo

```shellSession
git clone git@github.com:zalando/postgres-operator.git
git checkout c9acd527
```

### Configure nessecary variables

```shellSession
export aws_region=""            # S3 service region
export aws_endpoint=""          # Endpoint to S3 service
export aws_access_key_id=""     # Access key to S3 service
export aws_secret_access_key="" # Secret key to S3 service
export wal_s3_bucket=""         # S3 bucket to store wal and backups

# If your S3 provider only supports path style access. 
export aws_s3_force_path_style="true"
```

### Install wal-g environment secret

```shellSession
mkdir ignore && cp manifests/walg-environment-secret.yaml ignore/walg-environment-secret.yaml

yq_4 e '(.stringData.AWS_REGION, .stringData.CLONE_AWS_REGION, .stringData.STANDBY_AWS_REGION) |= env(aws_region)' -i ignore/walg-environment-secret.yaml
yq_4 e '(.stringData.AWS_ENDPOINT, .stringData.CLONE_AWS_ENDPOINT, .stringData.STANDBY_AWS_ENDPOINT) |= env(aws_endpoint)' -i ignore/walg-environment-secret.yaml
yq_4 e '(.stringData.AWS_ACCESS_KEY_ID, .stringData.CLONE_AWS_ACCESS_KEY_ID, .stringData.STANDBY_AWS_ACCESS_KEY_ID) |= env(aws_access_key_id)' -i ignore/walg-environment-secret.yaml
yq_4 e '(.stringData.AWS_SECRET_ACCESS_KEY, .stringData.CLONE_AWS_SECRET_ACCESS_KEY, .stringData.STANDBY_AWS_SECRET_ACCESS_KEY) |= env(aws_secret_access_key)' -i ignore/walg-environment-secret.yaml
[ "${aws_s3_force_path_style}" = "true" ] && yq_4 e '(.stringData.AWS_S3_FORCE_PATH_STYLE, .stringData.CLONE_AWS_S3_FORCE_PATH_STYLE, .stringData.STANDBY_AWS_S3_FORCE_PATH_STYLE) |= env(aws_s3_force_path_style)' -i ignore/walg-environment-secret.yaml || true

kubectl apply -f ignore/walg-environment-secret.yaml
```

### Install operator

We'll be using the [CRD-based configuration](https://github.com/zalando/postgres-operator/blob/master/docs/reference/operator_parameters.md#configuration-parameters).

```shellSession
helm upgrade --install postgres-operator ./postgres-operator/charts/postgres-operator \
    -f ./postgres-operator/charts/postgres-operator/values-crd.yaml \
    --set image.tag=v1.6.1-6-gc9acd527 \
    --set configKubernetes.spilo_privileged=true \
    --set configMajorVersionUpgrade.major_version_upgrade_mode=manual \
    --set configKubernetes.pod_environment_secret=walg-env \
    --set configAwsOrGcp.wal_s3_bucket=${wal_s3_bucket}
```

Confirm that operator is running.

```shellSession
kubectl get pod -l app.kubernetes.io/name=postgres-operator
```

If it's not you can `describe` the pod or check the logs.

```shellSession
kubectl describe "$(kubectl get pod -l app.kubernetes.io/name=postgres-operator --output='name')"

kubectl logs "$(kubectl get pod -l app.kubernetes.io/name=postgres-operator --output='name')"
```


### Install postgres

```shellSession
kubectl apply -f manifests/main.yaml
```

Watch pods being created
```shellSession
kubectl get pods -l application=spilo,cluster-name=iam-main -L spilo-role -w
```

Check logs of master.
```shellSession
kubectl logs $(kubectl get pods -l spilo-role=master,cluster-name=iam-main --output='name') -f -c postgres
```

Check operator logs.
```shellSession
kubectl logs "$(kubectl get pod -l app.kubernetes.io/name=postgres-operator --output='name')" -f
```


### Add dummy data and workload
We'll be using sysbench to generate some data and load against the servers.


Run job to initialize test data.
```shellSession
kubectl apply -f manifests/sysbench-job.yaml
```

Check that the job gets Completed.
```shellSession
kubectl get pods -l job-name=sysbench-prepare -w
```

Start workload.
```shellSession
kubectl apply -f manifests/sysbench-r.yaml -f manifests/sysbench-rw.yaml
```

Check out from workloads.
```shellSession
kubectl logs -l app=sysbench -f
```


### Clone from S3 with major version upgrade

Check current version that the main cluster is running.
```shellSession
kubectl exec -it $(kubectl get pods -l spilo-role=master --output='name') -c postgres -- su postgres -c "psql -c 'select version();' && psql --version"
```

Prepare clone manifest.
You will have to manually decide the recovery target time.

```shellSession
cp manifests/clone-s3.yaml ignore/
export iam_main_uid=$(kubectl get postgresql iam-main -o 'jsonpath={.metadata.uid}')
yq_4 e '.spec.clone.uid |= env(iam_main_uid)' -i ignore/clone-s3.yaml

# Example recovery target time
yq_4 e '.spec.clone.timestamp = "2021-03-29T15:44:00+01:00"' -i ignore/clone-s3.yaml
```

Install cluster.
```shellSession
kubectl apply -f ignore/clone-s3.yaml
```

Check logs.
```shellSession
kubectl logs iam-s3-clone-0 -f -c postgres
```

```shellSession
kubectl exec -it $(kubectl get pods -l spilo-role=master,cluster-name=iam-s3-clone --output='name') -c postgres -- su postgres -c "psql -c 'select version();'"
```


### In-place major version upgrade

Check current version.
```shellSession
kubectl exec -it $(kubectl get pods -l spilo-role=master --output='name') -c postgres -- su postgres -c "psql -c 'select version();' && psql --version"
```

Patch cluster with newer major version.
```shellSession
kubectl patch postgresql iam-main -p '{"spec":{"postgresql":{"version":"13"}}}'
```

Check operator logs.
```shellSession
kubectl logs "$(kubectl get pod -l app.kubernetes.io/name=postgres-operator --output='name')" -f
```

Check version.
```shellSession
kubectl exec -it $(kubectl get pods -l spilo-role=master --output='name') -c postgres -- su postgres -c "psql -c 'select version();' && psql --version"
```


### Standby cluster

```shellSession
cp manifests/standby.yaml ignore/
yq_4 e '.spec.standby.s3_wal_path = "s3://" + env(wal_s3_bucket) + "/spilo/iam-main/" + env(iam_main_uid) + "/wal/13"' -i ignore/standby.yaml
kubectl apply -f ignore/standby.yaml
```

Check logs.
```shellSession
kubectl logs iam-standby1-0 -f -c postgres
```


### Cleanup

```shellSession
kubectl delete postgresql --all
kubectl delete jobs --all
kubectl delete pod -l app=sysbench
```


## Notes

### Encryption

The operator/spilo does not support easy configuration of pgp encryption.
I think I managed to hack it together somewhat for a new cluster, but it breaks bootstrapping from cloning so it's not usable.
The main problem is that it not possible to configre wal-g environemtn with arbitrary wal-g environment variables.

### Exposing psql

I don't know how one would expose the postgresql pods.
There's this note about it in the [NGINX Ingress Controller docs](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/).
I have not tried that though.

### Logical backup

I haven't tried it yet.


### 
