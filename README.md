# Kylin on GCP with shell command only

This demo uses only cloud shell command, not terraform as initial project.

This project demonstrates the use of [Apache Kylin](http://kylin.apache.org/) on
[GCP](https://cloud.google.com/gcp/) backed by [Dataproc](https://cloud.google.com/dataproc/).

Forked project from muvaki : [muvaki kylin-gcp](https://github.com/muvaki/kylin-gcp/)

## Prerequisites
To get started you will need a GCP project, to create one see [here](https://cloud.google.com/resource-manager/docs/creating-managing-projects).

HBase version used : 1.3.5

Kylin version used : 2.6.2

[link for Apache Kylin version](https://mirror.csclub.uwaterloo.ca/apache/kylin/)

[link for HBase version](https://www-us.apache.org/dist/hbase/)

If the version I used are no longer available on those links, change the value in `/init-actions/hbase.sh` or `/init-actions/kylin.sh` by one of the available one in links above.

## Preparation for the cluster creation

To create the GCS bucket using the cloud shell :
```sh
gsutil mb -c regional -l REGION gs://BUCKET_NAME-res-bucket
```
I used `europe-west1` as `REGION`. Be aware that everybody can have access to your bucket so don't hesitate to name it randomly to make it harder for others.


Go to your cluster you just created in GCS and drag and drop the `init-actions` folder (and its content)


### Create Hadoop/Kylin cluster

To create the GCP dataproc cluster :
```sh
export bucket_name=YOUR_BUCKET_NAME
gcloud dataproc clusters create "kylin-cluster" \
	--region "europe-west1" \
	--zone "europe-west1-d" \
	--num-masters=1 \
	--num-workers=2 \
	--master-machine-type=n1-standard-2 \
	--worker-machine-type=n1-standard-1 \
	--bucket=$bucket_name \
	--initialization-actions "gs://$bucket_name/init-actions/hbase.sh,gs://$bucket_name/init-actions/kylin.sh,gs://$bucket_name/init-actions/hive-hcatalog.sh,,gs://$bucket_name/init-actions/hue.sh"
```

If successful, the deployment will deploy a resource bucket and upload the
necessary `init-scripts`, then create the Kylin cluster.

Once the cluster is up and running, start Kylin on one of the nodes; note a
master node is being used in this example, however any node can be seleted:
```sh
gcloud compute ssh ${MASTER} --command="source /etc/profile && kylin.sh start"
```
This will start the Kylin service. With Kylin installed and running yo can now
tunnel to the master node to bring up the kylin UI:
```sh
gcloud compute ssh ${MASTER} -- -L 7070:${MASTER}:7070
# then open browser to http://localhost:7070/kylin
# default creds ADMIN/KYLIN
```

### Writting data to cluster
Kylin gets its data from structured sources such as [Hive](https://hive.apache.org/)
and other JDBC/ODBC compliant sources. Hive in turn relies on the Hadoop
filesystem [HDFS](https://hadoop.apache.org/) to store the unerlying data.
In standard (on-prem) Hadoop deployments, HDFS would be deployed accross the
disks attached to the workers of the Hadoop cluster. In GCP Dataproc however,
the HDFS filesystem can (and is by default) backed by the GCS object storage
service using the [Cloud Storage Connector](https://cloud.google.com/dataproc/docs/concepts/connectors/cloud-storage).
This provides the decoupling of storage and compute resources, providing the
cost-benefits of both cheaper/scalable storage in comparison to persistant disks,
and allows cluster nodes (or the at least the workers) to be ephemiral because
they are only used for computation and not for persistent storage.

Building on the former, Hive can be used by creating table/schema definitions
and then pointing them to "directories" in HDFS (GCS). Once a table is created,
data can be written to GCS from any source using the standard GCS APIs without
the need to use any hive/hadoop interfaces. To demonstrate this, create a table
with a schema pointing to a path in GCS:
```sql
# example creation of hive table
hive -e "CREATE EXTERNAL TABLE test
(
unique_key STRING,
complaint_type STRING,
complaint_description STRING,
owning_department STRING,
source STRING,
status STRING,
status_change_date TIMESTAMP,
created_date TIMESTAMP,
last_update_date TIMESTAMP,
close_date TIMESTAMP,
incident_address STRING,
street_number STRING,
street_name STRING,
city STRING,
incident_zip INTEGER,
county STRING,
state_plane_x_coordinate STRING,
state_plane_y_coordinate FLOAT,
latitude FLOAT,
longitude FLOAT,
location STRING,
council_district_code INTEGER,
map_page STRING,
map_tile STRING
)
row format delimited
fields terminated by ','
LOCATION 'gs://${BUCKET}/data/311/csv/' ;"

```
the above can be run on one of the dataproc nodes (`gcloud compute ssh ${MASTER}`)
or using `gcloud dataproc jobs submit ...`. Note that we are using CSV format
for this demo, however, there are many many options for serialization with
HDFS/Hive. for more info see [SerDe](https://cwiki.apache.org/confluence/display/Hive/SerDe).

To populate the table location path with data, run the following dataflow
job to which will extract public 311-service-request data from the city of
Austin, TX which is stored in BigQuery:
```sh
cd dataflow
./gradlew ronLocal # run Apache Beam job locally
# OR
./gradlew runFlow # submit job to run on GCP Dataflow
```
Note that this Beam job is simply extracting BigQuery data via query,
converting it to csv form, and writting it to GCS. Writting data to hive can
also be achieved in Beam using the [HCatalog IO](https://beam.apache.org/documentation/io/built-in/hcatalog/), however, is
not recommeded if the primary query engine is Kylin and not Hive.

As data is added to the path, new queries will pick up this new data:
```sh
hive -e "select count(unique_key) from test;"
```

With tables created and populated, return to the Kylin web interface; if the
session has terminated restart it with the tunnel:
```sh
gcloud compute ssh ${MASTER} -- -L 7070:${MASTER}:7070
# open browser to http://localhost:7070/kylin
# default creds ADMIN/KYLIN
```

From here, the table can be loaded in Kylin:
 - `Model` tab -> `Data Sources sub-tab`
 - `Load Table` (blue button)
 - enter table name "test"
 - Under `Tables` the hive database `DEFAULT` should be displayed with the `test` just loaded

Once tables are loaded you can continue to create models/cubes within the kylin
interface, for more see the [Kylin Docs](http://kylin.apache.org/docs/tutorial/create_cube.html).

## Cleanup
Deleting the Kylin cluster and associated resources:
```sh
gcloud dataproc clusters delete CLUSTER_NAME \
gsutil rm -r gs://BUCKET_NAME
```
OR delete the project entirely to ensure no other resources are incurring costs.

## Future Options:
- [ ] Secure cluster and access
- [ ] configure Kylin init-action for HA-mode (with load-balancing)
- [ ] Autoscaling setup
- [ ] BigTable (HBase) cube storage substitution
- [ ] persistant disk HDFS substitution
- [ ] Spark substitution
- [ ] Streaming cube (Kafka) sample
- [ ] JDBC source sample
- [ ] hadoop resource optimization (disk, cpu, preemptible workers, etc)
- [ ] cube creation sample (real-world data)
- [ ] [hive-metastore Cloud-SQL substition](https://cloud.google.com/solutions/using-apache-hive-on-cloud-dataproc)
