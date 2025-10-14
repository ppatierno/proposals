# Support for dynamic controller quorum

This proposal is about adding the support for the Apache Kafka dynamic controller quorum within the Strimzi Clusert Operator, in order to support controllers scaling.
The dynamic controller quorum was introduced since Kafka 3.9.0 via [KIP-853](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes).

## Current situation

Currently, the Strimzi Cluster Operator supports the KRaft static controller quorum only, despite the Apache Kafka 4.x releases have the support for the dynamic one.
The static controller quorum doesn't allow scaling the controller nodes without downtime.
It means that after the Apache Kafka cluster creation, when the controller nodes are deployed, it is not possible to add or remove controllers from the quorum anymore, unless accepting some downtime for the Apache Kafka cluster.

The only possible way for scaling controllers is:

* pause the cluster reconciliation
* delete the controllers' `StrimziPodSet`(s) so that the controller pods are deleted
* update the number of replicas within the `KafkaNodePool` custom resource related to controllers (increase or decrease)
* unpause the cluster reconciliation

The Strimzi Cluster Operator will be able to create the new controllers pool from scratch (with the updated number of replicas) and it will restart the brokers with the new controller quorum configured (via `controller.quorum.voters` property).
Of course, right after the controllers deletion step and until the controller quorum runs again, the cluster won't be available because brokers cannot work without the KRaft controllers running.

This is not parity features with ZooKeeper-based cluster where the ZooKeeper ensemble can be scaled up or down without cluster downtime.

## Motivation

Dynamic controller quorum allows to enable the KRaft controllers quorum scaling without downtime.
This would provide parity features with a ZooKeeper-based cluster with ZooKeeper nodes can be added or removed to the quorum.

TBD

## Proposal

TBD

TODO list:

* add support for dynamic controller quorum to the Strimzi Cluster Operator
    * using it by default for any newly created Apache Kafka cluster? ...
    * ... or behind a feature gate?
* add support for controllers scaling when the dynamic controller quorum is used
* add migration from static to dynamic controller quorum

The `controller.quorum.voters` property is now replaced by the `controller.quorum.bootstrap.servers` property.

The `kraft.version` feature is 0 when using static quorum. It's greater than 0 (actually 1) when using dynamic quorum.
No need to set this feature upfront. It's being set when the controllers storage is formatted and the quorum is configured as dynamic or static.

Scaling up more than one controllers at time should be avoided (Disjoint majorities problem).
Only one controller at time should be added.
Multi-controllers scaling should be done as a sequence of one controller addition at time.
A new controller is added as "observer" first, right after the creation.
Only when it's in sync with the other controllers, it can be added to the quorum.

Scaling down is always done one controller at time.
It's about unregistering the controller and then killing the pod.

TODO: should it be a validation check within the Strimzi Cluster Operator? A `KafkaNodePool` hosting controllers should be scaled up/down only by one (replicas field).

### The storage formatting "challenge"

Currently, when it comes to formatting the node storage (broker or controller) during the node startup, the `kafka-storage.sh` tool is used within the `kafka_run.sh` script with the following parameters:

* the cluster ID (with `-t` option)
* the metadata version (with `-r` option)
* the path to the configuration file (with `-c` option)

It also adds the `-g` option in order to ignore the formatting if the storage is already formatted.
Such option is needed because the same tool is executed every time on node startup (i.e. even during a rolling) when the node storage is already formatted.
So the current command is the following:

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g
```

With the `STRIMZI_CLUSTER_ID` and `METADATA_VERSION` variable coming from corresponding fields within the ConfigMap which is generated for each node and mounted on the pod volume.

This works when the Apache Kafka cluster is using the static quorum with during a new cluster creation, when all nodes need to be formatted from scratch at the same time, as well as on cluster scaling, when a new broker node is added but the formatting is done the same way (of course controller scaling is not supported).

The dynamic quorum works differently from the formatting perspective and it makes a clear distintion on the `kafka-storage.sh` usage when it's a new cluster creation or a node addition (both broker or controller).
When the cluster is created, each broker can be formatted the same way we do today but the controller formatting needs the new `-I` option which specifies the initial controllers list.
The initial controllers list is a comma separated list of controllers, as `node-id@dns-name:port:directory-id`, which contains the initial controllers building the KRaft quorum in the new cluster we are going to create.

The formatting tool is then used the following way:

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -I="$INITIAL_CONTROLLERS"
```

The issue on cluster creation is about differentiating between broker and controller to use the right options for the formatting.
The broker storage formatting doesn't have the `-I` option.
But when it comes to scaling up, so adding a new broker or a controller on an existing cluster, in both cases the formatting tool needs the `-N` (no initial controllers) instead, which replace the `-I` in case of a controller.

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -N
```

So it means that when using the dynamic quorum, the script executed on node startup needs a way to differentiate between a new cluster creation or a node scaling in order to run the formatting with the appropriate options.

The above scenarios can be summarized this way:

* Static quorum
  * Broker and controllers are formatted with no additional options
* Dynamic quorum
  * New cluster creation:
    * broker doesn't need any additional options (but it works well with `-N` as well to simplify the proposal, see later)
    * controller needs the initial controllers list via `-I`
  * Existing cluster, scale up:
    * broker and controller needs the `-N` option

The proposed idea is about having the Strimzi Cluster Operator generating the initial controllers list, which includes the directory IDs generation, that can be passed through the mounted ConfigMap to the run script to be used during the formatting.
The initial controller list would be used as a differentiator between using static quorum (the list is empty) or dynamic quorum (the list is filled).
The script can also get the role of the current node as broker and/or controller by reading it from the `process.roles` field within the `/tmp/strimzi.properties` file.

The initial controllers list is also stored in a new `initialControllers` field within the `Kafka` custom resource `status` which is set on cluster creation with dynamic quorum.
It will be enough storing controller node and directory IDs the following way:

```yaml
# ...
status:
    # ...
    initialControllers:
    - directoryId: 5veSrHXcRYq5pq-13P0eDg
      nodeId: 6
    - directoryId: Jx3Y7QvUTQK4rqPGOCq6pA
      nodeId: 7
    # ...
```

Having this field set (or not) is also a way to distinguish the reconciliation of an existing Kafka cluster using the new dynamic quorum (the field is set) or still using the static quorum (the field is missing).
This way the Strimzi Cluster Operator, together with the run script on the node, can handle both static and dynamic quorum based clusters and proper node formatting.

More details about the initial controllers list creation in the following sections.

### New cluster creation with dynamic quorum

The initial cluster creation is going to use the [bootstrap with multiple controllers](https://kafka.apache.org/documentation/#kraft_nodes_voters) approach.
The Strimzi Cluster Operator:

* builds the broker and controller configuration by setting the `controller.quorum.bootstrap.servers` field (instead of the `controller.quorum.voters` one).
* generates a random directory ID, as a UUID (like the cluster ID), for each controller.
    * controllers' directory IDs are saved within the `Kafka` status. The directory ID for a specific controller is needed during the removal operation (together with the node ID).
* builds the initial controllers list, as `initial.controllers` field within the node ConfigMap, to be loaded by the `kafka_run.sh` script where it's needed for formatting the storage properly.
* saves the `initialControllers` field within the `Kafka` custom resource status by storing the `nodeId` and `directoryId` for each initial controller.

On node start up, within the `kafka_run.sh` script, the `initial.controllers` field is loaded from the node ConfigMap into a `INITIAL_CONTROLLERS` variable:

* if the variable is empty, the cluster is static quorum based, so using the usual formatting as today.
* if the variable is not empty, the cluster is dynamic quorum based, and the script gets the node role into a `PROCESS_ROLES` variable.
  * if it's a broker, formatting with the `-N` option (which works for both new cluster creation or brokers scale up)
  * if it's a controller, the script check if it's part of the initial controllers list or not:
    * part of the initial controller list, formatting with the `-I` option
    * not part of the initial controller list, it's a controller scale up, formatting with the `-N` option

Of course when we have a mixed node, because the `process.roles` includes being a controller, the run script follows the corresponding controller path for formatting.

### Reconcile an existing cluster with static quorum

When the Strimzi Cluster Operator is upgraded with the latest version supporting dynamic quorum, it's going to reconcile existing `Kafka` custom resources for clusters which are using the static quorum.
The operator detects it's not the creation of a new cluster and also that the `initialControllers` status field is not filled within the `Kafka` custom resources.
This is a way to understand that the existing cluster is not new and it's using the static quorum.
In such a case the reconciliation proceed as usual:

* builds the broker and controller configuration by setting the `controller.quorum.voters` field.
* doesn't build the `initial.controllers` field within the node ConfigMap.

It means that there is no automatic migration from static to dynamic quorum.
The Strimzi Cluster Operator will be able to detect which type of quourm (static vs dynamic) an existing cluster is using without modifying it but just going through the proper reconciliation process.

### Adding a new controller (scale up)

When the node pool hosting the controllers is scaled up, in order to add a controller, the Strimzi Cluster Operator:

* validates that the scaling up was just by one because it's the recommended way to scaling controllers by the official Apache Kafka documentation. Any attempts to scale up by more than one controller in one step is declined.
* builds the new controller configuration by setting the `controller.quorum.bootstrap.servers` field (instead of the `controller.quorum.voters` one).

On node start up, within the `kafka_run.sh` script:

* the controller node is formatted by using the `-N` option.

NOTE: It seems to be ok used for a broker as well, not just controller as per official Kafka documentation  [here](https://kafka.apache.org/documentation/#kraft_nodes_observers).

Back to the Strimzi Cluster Operator:

* monitoring that the new controller has caught up with the active ones in the quorum.
* add/register the controller within the quorum by using the `addRaftVoter` method in the Kafka Admin API.

TBD

Manual approach after scaling up to move the controller from being an observer to join the quorum and become a voter.

```shell
bin/kafka-metadata-quorum.sh --command-config /tmp/strimzi.properties --bootstrap-server my-cluster-kafka-bootstrap:9092 add-controller
```

Checking that the controller was "promoted" to voter by using the following command:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
```

### Removing a controller (scale down)

When the node pool hosting the controllers is scaled down, in order to remove a controller, the Strimzi Cluster Operator:

* removes/unregister the controller from the quorum by using the `removeRaftVoter` method in the Kafka Admin API.
* delete the controller pod.

The above sequence is possible, with [KIP-996](https://cwiki.apache.org/confluence/display/KAFKA/KIP-996%3A+Pre-Vote) in place since Apache Kafka 4.0.0.

TODO: double check if the above is true or we need to do the opposite (kill pod first, then unregister controller). Depends on KIP-996 which should be available since Kafka 4.0.0.

TBD

Manual approach to remove the controller to come back being an observer (not voter anymore) before scaling down:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 remove-controller --controller-id <id> --controller-directory-id <directory-id>
```

with `<id>` as the controller node id and the `<directory-id>` to be retrieved from the controller node in the `meta.properties` file.
After scaling down the controller, it will be still showed as observer for some time until (I guess) a voters cache is refreshed.
Checking that the controller was removed from the observers list by using the following command:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
```

### Migration from static to dynamic quorum

Migration from static to dynamic quorum is supported starting from Apache Kafka 4.1.0.
The plan should be having the overall dynamic quorum support (together with migration) in a Strimzi release with Apache Kafka 4.1.0 as the minimum version.
The procedure is available on the official Kafka documentation [here](https://kafka.apache.org/documentation/#kraft_upgrade).

* the `kraft.version` has to be set to `1` (greater than 0 anyway).
* the `controller.quorum.bootstrap.servers` field should be used instead of the `controller.quorum.voters`.

The Strimzi Cluster Operator should be also able to build the `initialControllers` field:

* by reading the controller IDs from the `controller.quorum.voters` (they are going to be the initial controllers)
* getting the corresponding directory IDs by using Kafka Admin Client API `describeMetadataQuorum` method, and extract from the `QuorumInfo` (the `ReplicaState.replicaDirectoryId()` for each node within the `voters` field).

The `initialControllers` list is then saved within the `Kafka` custom resource status.

TBD

Manual approach:

* ssh into a node and run

```shell
bin/kafka-features.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 upgrade --feature kraft.version=1
```
* ssh into controllers, get all the directory IDs from the `meta.properties` files patch the corresponding `Kafka` status.

```shell
kubectl patch kafka my-cluster -n myproject --type=merge --subresource=status -p '{"status":{"initialControllers":[{"nodeId":3,"directoryId":"aIbghWckQcCqhuwcH-Im_Q"},{"nodeId":4,"directoryId":"mfxmiCJjQhmis2FkPZ6-aw"},{"nodeId":5,"directoryId":"I7BRhBWqSqSTgi1WG9KEuA"}]}}'
```

Patching the `Kafka` custom resource status will trigger nodes rolling and the operator will reconfigure them with the `controller.quorum.bootstrap.servers` field for using the dynamic quorum.

NOTE: it's not possible to use the `kafka-metadata-quorum.sh` tool with `--replication` to get the directory IDs because, if the cluster is using static quorum, it will return just `AAAAAAAAAAAAAAAAAAAAAA` and not the actual value.

## Affected/not affected projects

Only the Strimzi Cluster Operator.

## Compatibility

TBD

## Rejected alternatives

Storing the initial controllers with the corresponding node and directory IDs within the `KafkaNodePool` instead of the `Kafka` custom resource.
The initial controllers list is a global information that should be placed into one place, the `Kafka` custom resource, because it belongs to the entire cluster.
Spreading this information across several custom resources (because you can have more than one node pools hosting `controllers`) would add not needed complexity.
