# postgres-in-k8s
POC using Zalando's [PostgreSQL K8s Operator](https://github.com/zalando/postgres-operator).


## Requirements

- A running Kubernetes Cluster.
    - Priviledged podSecurityPolicy installed.
- Helm v3.
- Access to an S3 service.
- Yq version 4.x, named `yq_4`.


#### Get Zalando's postgres-operator repo

```shellSession
git clone git@github.com:zalando/postgres-operator.git
git checkout c9acd527
```

#### Configure nessecary variables

```shellSession
export aws_region=""            # S3 service region
export aws_endpoint=""          # Endpoint to S3 service
export aws_access_key_id=""     # Access key to S3 service
export aws_secret_access_key="" # Secret key to S3 service
export wal_s3_bucket=""         # S3 bucket to store wal and backups

# If your S3 provider only supports path style access. 
export aws_s3_force_path_style="true"
```

#### Install wal-g environment secret

```shellSession
mkdir ignore && cp manifests/walg-environment-secret.yaml ignore/walg-environment-secret.yaml

yq_4 e '(.stringData.AWS_REGION, .stringData.CLONE_AWS_REGION, .stringData.STANDBY_AWS_REGION) |= env(aws_region)' -i ignore/walg-environment-secret.yaml
yq_4 e '(.stringData.AWS_ENDPOINT, .stringData.CLONE_AWS_ENDPOINT, .stringData.STANDBY_AWS_ENDPOINT) |= env(aws_endpoint)' -i ignore/walg-environment-secret.yaml
yq_4 e '(.stringData.AWS_ACCESS_KEY_ID, .stringData.CLONE_AWS_ACCESS_KEY_ID, .stringData.STANDBY_AWS_ACCESS_KEY_ID) |= env(aws_access_key_id)' -i ignore/walg-environment-secret.yaml
yq_4 e '(.stringData.AWS_SECRET_ACCESS_KEY, .stringData.CLONE_AWS_SECRET_ACCESS_KEY, .stringData.STANDBY_AWS_SECRET_ACCESS_KEY) |= env(aws_secret_access_key)' -i ignore/walg-environment-secret.yaml
[ "${aws_s3_force_path_style}" = "true" ] && yq_4 e '(.stringData.AWS_S3_FORCE_PATH_STYLE, .stringData.CLONE_AWS_S3_FORCE_PATH_STYLE, .stringData.STANDBY_AWS_S3_FORCE_PATH_STYLE) |= env(aws_s3_force_path_style)' -i ignore/walg-environment-secret.yaml || true

kubectl apply -f ignore/walg-environment-secret.yaml
```

#### Install operator

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


#### Install postgres

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


#### Add dummy data and workload
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


#### Clone from S3 with major version upgrade

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


#### In-place major version upgrade

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


#### Standby cluster

```shellSession
cp manifests/standby.yaml ignore/
yq_4 e '.spec.standby.s3_wal_path = "s3://" + env(wal_s3_bucket) + "/spilo/iam-main/" + env(iam_main_uid) + "/wal/13"' -i ignore/standby.yaml
kubectl apply -f ignore/standby.yaml
```

Check logs.
```shellSession
kubectl logs iam-standby1-0 -f -c postgres
```


#### Cleanup

```shellSession
kubectl delete postgresql --all
kubectl delete jobs --all
kubectl delete pod -l app=sysbench
```


## Notes

#### Encryption

The operator/spilo does not support (easy) configuration of pgp encryption.
I think I managed to hack it together somewhat for a new cluster, but it breaks bootstrapping from cloning so it's not usable.
The main problem is that it not possible to configre wal-g environemtn with arbitrary wal-g environment variables.

#### Exposing psql

I don't know how one would expose the postgresql pods.
There's this note about it in the [NGINX Ingress Controller docs](https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/).

#### Logical backup

I haven't tried it yet.
Still the same problem as before that you specify AWS credentials directly in [helm values](https://github.com/zalando/postgres-operator/blob/master/docs/reference/operator_parameters.md#logical-backup) instead of through secrets.
https://github.com/zalando/postgres-operator/issues/1247

There is no way of boostrapping a new postgresql cluster from a logical backup.
This is a procedure that one would have to figure out and document.

#### Logs

It would take some work to get some decent logs to Elasticsearch from the pods.
The [ingest pipeline](https://github.com/elastic/beats/blob/master/filebeat/module/postgresql/log/ingest/pipeline.yml) is really only meant for postgresql logs.

```shellSession
# wal-g
INFO: 2021/04/12 05:35:36.946207 FILE PATH: 0000000200000036000000AB.br

# pgq_ticker
2021-04-12 05:35:35,140 INFO: Lock owner: iam-main-0; I am iam-main-0
2021-04-12 05:35:35,213 INFO: no action.  i am the leader with the lock

# postgresql
2021-04-12 05:33:36 UTC [614]: [5198-1] 607024a0.266 116812188 sbtest sbtest_owner_user [unknown] 10.233.77.179 ERROR:  duplicate key value violates unique constraint "sbtest8_pkey"
2021-04-12 05:33:36 UTC [614]: [5199-1] 607024a0.266 116812188 sbtest sbtest_owner_user [unknown] 10.233.77.179 DETAIL:  Key (id)=(4987) already exists.
```

#### Storage

A consideration that one has to make before running in PostgreSQL in Kubernetes (and in the cloud for that matter), is to determine what requirements the clusters will need on the underlying storage.
- Is block storage integration configured in the Kubernetes cluster?
    - If, so what performance does each available storageclass give you?

If you come to the conclusion that block storage does not give you, let's say, sufficient read/write latency, you might have to consider using local pv's in Kubernetes which forces the pods to be running on specific Kubernetes nodes, thus eliminating some of the freedom that Kubernetes originally offers and safety that block storage gives you.

#### The operator is a piece of code
... that naturally contains and can introduce bugs that might be quite detremental to your clusters.
The maintainers seems to be address bugs in a quick manner and release patch version if the bug is effecting enough people.
One such recent bug is the one fixed by [this pr](https://github.com/zalando/postgres-operator/pull/1380).

## Related reading

#### Patroni

- https://patroni.readthedocs.io/en/latest/index.html

#### Spilo

- https://github.com/zalando/spilo
- https://github.com/zalando/spilo/tree/master/postgres-appliance/scripts

#### Postgres operator
- https://postgres-operator.readthedocs.io/en/latest/