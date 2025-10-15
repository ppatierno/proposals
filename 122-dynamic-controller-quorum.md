# Support for dynamic controller quorum

This proposal is about adding the support for the Apache Kafka dynamic controller quorum within the Strimzi Clusert Operator, in order to support controllers scaling as well.
The dynamic controller quorum was introduced since Kafka 3.9.0 via [KIP-853](https://cwiki.apache.org/confluence/display/KAFKA/KIP-853%3A+KRaft+Controller+Membership+Changes).

## Current situation

Currently, the Strimzi Cluster Operator supports the KRaft static controller quorum only, despite the Apache Kafka 4.x releases have the support for the dynamic one.
The static controller quorum doesn't allow scaling the controller nodes without downtime.
It means that after the Apache Kafka cluster creation, when the controller nodes are deployed, it is not possible to add or remove controllers from the quorum anymore, unless accepting some downtime for the Apache Kafka cluster.

The only possible way for scaling controllers is:

* pause the cluster reconciliation.
* delete the controllers' `StrimziPodSet`(s) so that the controller pods are deleted.
* update the number of replicas within the `KafkaNodePool`(s) custom resource(s) related to controllers (increase or decrease).
* unpause the cluster reconciliation.

The Strimzi Cluster Operator will be able to create the new controllers from scratch (with the updated number of replicas) and it will restart the brokers with the new controller quorum configured (via `controller.quorum.voters` property).
Of course, right after the controllers deletion step and until the controller quorum runs again, the cluster won't be available because brokers cannot work without the KRaft controllers running.

This is not parity features with ZooKeeper-based cluster where the ZooKeeper ensemble can be scaled up or down without cluster downtime.

## Motivation

Dynamic controller quorum allows to enable the KRaft controllers quorum scaling without downtime.
This would provide parity features with a ZooKeeper-based cluster with ZooKeeper nodes can be added or removed to the quorum.

TBD

## Proposal

This proposal is about supporting the dynamic quorum and controllers scaling within the Strimzi Cluster Operator, by going through three main steps:

* Add support for dynamic controller quorum to the Strimzi Cluster Operator and using it by default for any newly created Apache Kafka cluster.
* Add support for controllers scaling when the dynamic controller quorum is used.
* Add migration from static to dynamic controller quorum.

It also take into account the main operational changes when moving from static to dynamic quorum:

* Node storage formatting is impacted by this change.
* The `controller.quorum.voters` property is now replaced by the `controller.quorum.bootstrap.servers` property.
* The `kraft.version` feature is now 1 when using dynamic quorum. It's 0 when using static quorum. By the way, there is no need to set this feature upfront. It's being set when the controllers storage is formatted and the quorum is configured as dynamic or static. The only case needing the feature to be changed is when migrating from static to dynamic quorum (so updating the value from 0 to 1).

The following sections describes what's the idea behind the proposal and how to approach the above steps.
As a referece, [here](https://kafka.apache.org/documentation/#kraft_nodes)'s the link to the official Apache Kafka documentation about using dynamic quorum.

### The storage formatting "challenge"

Currently, when it comes to formatting the node storage (broker or controller) during the node startup, the `kafka-storage.sh` tool is used within the `kafka_run.sh` script with the following parameters:

* the cluster ID (with `-t` option)
* the metadata version (with `-r` option)
* the path to the configuration file (with `-c` option, pointing to `/tmp/strimzi.propertis` on the pod)

It also adds the `-g` option in order to ignore the formatting if the storage is already formatted.
Such option is needed because the same tool is executed every time on node startup (i.e. even during a rolling) when the node storage is already formatted.
So the current command is the following:

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g
```

With the `STRIMZI_CLUSTER_ID` and `METADATA_VERSION` variable coming from corresponding fields within the ConfigMap which is generated for each node and mounted on the pod volume.

This works when the Apache Kafka cluster is using the static quorum during a new cluster creation, when all nodes need to be formatted from scratch at the same time, as well as on cluster scaling, when a new broker node is added but the formatting is done the same way (of course controller scaling is not supported).

The dynamic quorum works differently from the formatting perspective and it makes a clear distintion on the `kafka-storage.sh` usage when it's a new cluster creation or a node addition (both broker or controller).
When the cluster is created, each broker can be formatted the same way we do today but the controller formatting needs the new `-I` (or `--initial-controllers`) option which specifies the initial controllers list.
The initial controllers list is a comma separated list of controllers, as `node-id@dns-name:port:directory-id`, which contains the initial controllers building the KRaft quorum in the new cluster we are going to create.

The formatting tool is then used the following way:

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -I="$INITIAL_CONTROLLERS"
```

The issue on cluster creation is about differentiating between broker and controller to use the right options for the formatting.
The broker storage formatting doesn't use the `-I` option.
But when it comes to scaling up, so adding a new broker or a controller on an existing cluster, in both cases the formatting tool needs the `-N` (or `--no-initial-controllers`) option instead, which replace the `-I` in case of a controller.

```shell
./bin/kafka-storage.sh format -t="$STRIMZI_CLUSTER_ID" -r="$METADATA_VERSION" -c=/tmp/strimzi.properties -g -N
```

So it means that when using the dynamic quorum, the script executed on node startup needs a way to differentiate between a new cluster creation or a node scaling in order to run the formatting with the appropriate options.

The above scenarios can be summarized this way:

* Static quorum:
  * Broker and controllers are formatted with no additional new options.
* Dynamic quorum:
  * New cluster creation:
    * broker doesn't need any additional options (but it works well with `-N` as well to simplify the proposal, see later).
    * controller needs the initial controllers list via `-I`.
  * Existing cluster, scale up:
    * broker and controller needs the `-N` option.

The proposed idea is about having the Strimzi Cluster Operator generating the initial controllers list, which includes the directory IDs generation, that can be passed through the mounted ConfigMap, via a new `initial.controllers` field, to the run script to be used during the formatting, via a corresponding `INITIAL_CONTROLLERS` variable.
The initial controller list would be used as a differentiator between using static quorum (the list is empty) or dynamic quorum (the list is not empty).
The script can also get the role of the current node, broker and/or controller, by reading it from the `process.roles` field within the `/tmp/strimzi.properties` file.

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

Of course, this field is not set when the cluster uses static quorum instead.
Having this field set (or not) is also a way to distinguish the reconciliation of an existing Kafka cluster using the new dynamic quorum (the field is set) or still using the static quorum (the field is missing).
This way the Strimzi Cluster Operator, together with the run script on the node, can handle both static and dynamic quorum based clusters and proper node formatting.

More details about the initial controllers list creation in the following sections.

### New cluster creation with dynamic quorum

The initial cluster creation is going to use the [bootstrap with multiple controllers](https://kafka.apache.org/documentation/#kraft_nodes_voters) approach.
The Strimzi Cluster Operator:

* builds the broker and controller configuration by setting the `controller.quorum.bootstrap.servers` field (instead of the `controller.quorum.voters` one).
* generates a random directory ID, as a UUID (like the cluster ID), for each controller.
* builds the initial controllers list, as `initial.controllers` field within the node ConfigMap, to be loaded by the `kafka_run.sh` script where it's needed for formatting the storage properly.
* saves the `initialControllers` field within the `Kafka` custom resource status by storing the `nodeId` and `directoryId` for each initial controller.

On node start up, within the `kafka_run.sh` script, the `initial.controllers` field is loaded from the node ConfigMap into a `INITIAL_CONTROLLERS` variable:

* if the variable is empty, the cluster is static quorum based, so using the usual formatting as today.
* if the variable is not empty, the cluster is dynamic quorum based, and the script gets the node role into a `PROCESS_ROLES` variable.
  * if it's a broker, formatting with the `-N` option (which works for both new cluster creation or brokers scale up).
  * if it's a controller, the script check if it's part of the initial controllers list or not:
    * part of the initial controller list, formatting with the `-I` option.
    * not part of the initial controller list, it's a controller scale up, formatting with the `-N` option.

Of course when we have a mixed node, because the `process.roles` includes being a controller, the run script follows the corresponding controller path for formatting.

### Reconcile an existing cluster with static quorum

When the Strimzi Cluster Operator is upgraded with the latest version supporting dynamic quorum, it's going to reconcile existing `Kafka` custom resources for clusters which are using the static quorum.
The operator detects it's not the creation of a new cluster and also that the `initialControllers` status field is not filled within the `Kafka` custom resources.
This is a way to understand that the existing cluster is not new and it's using the static quorum.
In such a case the reconciliation proceed as usual:

* builds the broker and controller configuration by setting the `controller.quorum.voters` field.
* doesn't build the `initialControllers` field within the node ConfigMap.

It means that there is no automatic migration from static to dynamic quorum.
The Strimzi Cluster Operator will be able to detect which type of quourm (static vs dynamic) an existing cluster is using without modifying it but just going through the proper reconciliation process.

### Adding a new controller (scale up)

Scaling up multiple controllers at the same time should be avoided to prevent disjoint majority issues (more details [here](https://developers.redhat.com/articles/2024/11/27/dynamic-kafka-controller-quorum)).
Controllers must be added one at a time, in sequence.
When performing multi-controller scaling, each addition should be completed before starting the next.
A newly added controller is initially created as an observer and once it is fully synchronized with the existing controllers, it can then be promoted to join the quorum.

When the node pool hosting the controllers is scaled up, in order to add a controller, the Strimzi Cluster Operator:

* validates that the scaling up was just by one because it's the recommended way to scaling controllers by the official Apache Kafka documentation. Any attempts to scale up by more than one controller in one step is declined.
* builds the new controller configuration by setting the `controller.quorum.bootstrap.servers` field (instead of the `controller.quorum.voters` one).

On node start up, within the `kafka_run.sh` script:

* the controller node is formatted by using the `-N` option.

Back to the Strimzi Cluster Operator:

* monitoring that the new controller has caught up with the active ones in the quorum.
* add/register the controller within the quorum by using the `addRaftVoter` method in the Kafka Admin API.

But monitoring that the controller has caught up with the active ones in the quorum could not be that easy.
Looking at log offsets and checking they are within a limit to say a controller is in sync can be complex and the root of several problems.

A different approach could be "promoting" the controller from being an observer to join the quorum and become a voter, by manually running the following command on the controller pod itself:

```shell
bin/kafka-metadata-quorum.sh --command-config /tmp/strimzi.properties --bootstrap-server my-cluster-kafka-bootstrap:9092 add-controller
```

Then checking that the controller was "promoted" to voter by using the following command:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
```

### Removing a controller (scale down)

Scaling down is always done one controller at time.
It's about unregistering the controller from the quorum and then killing the pod.

Based on [KIP-996](https://cwiki.apache.org/confluence/display/KAFKA/KIP-996%3A+Pre-Vote) in place since Apache Kafka 4.0.0, when the node pool hosting the controllers is scaled down, in order to remove a controller, the Strimzi Cluster Operator:

* removes/unregister the controller from the quorum by using the `removeRaftVoter` method in the Kafka Admin API.
* delete the controller pod.

In order to remove a controller, its directory ID is needed.
It can be retrieved by using the Kafka Admin Client API `describeMetadataQuorum` method, and extract the `ReplicaState.replicaDirectoryId()` for each node within the `voters` field from the `QuorumInfo`.

The corresponding manual approach instead, would be making the controller to come back being an observer (not voter anymore) before scaling down with the following command to execute on any node:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 remove-controller --controller-id <id> --controller-directory-id <directory-id>
```

with `<id>` as the controller node id and the `<directory-id>` to be retrieved from the controller node in the `meta.properties` file.

After scaling down the controller, it will be still showed as observer for some time until a voters cache is refreshed.
Then checking that the controller was removed from the observers list by using the following command:

```shell
bin/kafka-metadata-quorum.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 describe --status
```

### Migration from static to dynamic quorum

Migration from static to dynamic quorum is supported starting from Apache Kafka 4.1.0.
The plan should be having the overall dynamic quorum support (together with migration) in a Strimzi release with Apache Kafka 4.1.0 as the minimum version.
The procedure is available on the official Kafka documentation [here](https://kafka.apache.org/documentation/#kraft_upgrade) and it covers the following main steps:

* the `kraft.version` has to be set to `1` (greater than 0 anyway).
* the `controller.quorum.bootstrap.servers` field should be used instead of the `controller.quorum.voters`.

The Strimzi Cluster Operator should be also able to build the `initialControllers` field:

* by reading the controller IDs from the `controller.quorum.voters` (they are going to be the initial controllers)
* getting the corresponding directory IDs by using Kafka Admin Client API `describeMetadataQuorum` method, and extract from the `QuorumInfo` (the `ReplicaState.replicaDirectoryId()` for each node within the `voters` field).

The `initialControllers` list is then saved within the `Kafka` custom resource status.

But the step of retrieving the controllers' directory IDs don't work when we start from a static quorum.
The Kafka Admin Client API returns the `AAAAAAAAAAAAAAAAAAAAAA` value instead of the actual directory ID (from the `meta.properties` file).
It's the Base64 encoding of UUID all zeroes used within Apache Kafka when the voters are tracked within a static quorum.

Due to the above problem, a manual approach could be a solution by making the following steps:

* on any node from the cluster, running:

```shell
bin/kafka-features.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 upgrade --feature kraft.version=1
```
* going through all controllers, get all the directory IDs from the `meta.properties` files and patch the corresponding `Kafka` status:

```shell
kubectl patch kafka my-cluster -n myproject --type=merge --subresource=status -p '{"status":{"initialControllers":[{"nodeId":3,"directoryId":"aIbghWckQcCqhuwcH-Im_Q"},{"nodeId":4,"directoryId":"mfxmiCJjQhmis2FkPZ6-aw"},{"nodeId":5,"directoryId":"I7BRhBWqSqSTgi1WG9KEuA"}]}}'
```

Patching the `Kafka` custom resource status will trigger nodes rolling and the operator will reconfigure them with the `controller.quorum.bootstrap.servers` field for using the dynamic quorum.

NOTE: it's not possible to use the `kafka-metadata-quorum.sh` tool with `--replication` to get the directory IDs because, if the cluster is using static quorum, it will return just `AAAAAAAAAAAAAAAAAAAAAA` and not the actual value, because it's using the same Kafka Admin Client API as explained before.

## Affected/not affected projects

Only the Strimzi Cluster Operator.

## Compatibility

TBD

## Rejected alternatives

Storing the initial controllers with the corresponding node and directory IDs within the `KafkaNodePool` instead of the `Kafka` custom resource.
The initial controllers list is a global information that should be placed into one place, the `Kafka` custom resource, because it belongs to the entire cluster.
Spreading this information across several custom resources (because you can have more than one node pools hosting `controllers`) would add not needed complexity.
