# Kafka Commands

This document explains how to reach the Kafka command-line binaries on Windows, Linux/macOS, and inside the Kafka Docker container. It covers locating the binaries and execution of Kafka command.

## Command Line binaries path
### Windows

Kafka must be downloaded and extracted on Windows to use its Windows binaries. Docker Compose does not install these binaries on the Windows host.

1. Open PowerShell.
2. Navigate to the extracted Kafka distribution's `bin\windows` directory:

   ```powershell
   cd "<Kafka-folder>\bin\windows"
   ```

3. The Windows Kafka command scripts are in this directory and use the `.bat` extension.

### Linux or macOS

Kafka must be downloaded and extracted on the host to use its Linux/macOS binaries.

1. Open a terminal.
2. Navigate to the extracted Kafka distribution's `bin` directory:

   ```bash
   cd "<Kafka-folder>/bin"
   ```

3. The Linux/macOS Kafka command scripts are in this directory and use the `.sh` extension.

### Docker Compose container

The Kafka binaries are included in the Kafka container image. They are not available on the Windows host unless Kafka is separately installed there.

1. Open a shell in the running Kafka service:

   ```shell
   docker exec -it kafka /bin/sh
   ```
   `docker exec -it kafka /bin/sh` opens an interactive shell inside the running Kafka container.
   - `docker exec` runs a command in an existing container.
   - `-i` keeps input open; -t gives you a terminal.
   - `kafka` is the container name. List running containers and their names run `docker ps`
   - `/bin/sh` starts the shell.
     Use it to inspect or troubleshoot from inside the container. Type exit to leave; that doesn’t stop Kafka.

2. Inside the container, locate the directory containing Kafka's `bin` folder:

   ```shell
   find /opt -type d -name bin
   ```
   ```shell
   # Output like this
   /opt/kafka_2.13-2.8.1/bin
    ```

3. Change directory to the `bin` directory returned by the search for ex ``/opt/kafka_2.13-2.8.1/bin``. The exact path can vary with the Kafka image version.

4. The container is Linux-based, so its Kafka command scripts use the `.sh` extension.
5. Try ``ls`` to list all the binaries inside the ``bin`` directory. Ex listed below
```shell
connect-distributed.sh        kafka-cluster.sh           kafka-consumer-perf-test.sh  kafka-leader-election.sh             kafka-producer-perf-test.sh    kafka-server-stop.sh                kafka-verifiable-producer.sh     zookeeper-server-stop.sh
connect-mirror-maker.sh       kafka-configs.sh           kafka-delegation-tokens.sh   kafka-log-dirs.sh                    kafka-reassign-partitions.sh   kafka-storage.sh                    trogdor.sh                       zookeeper-shell.sh
connect-standalone.sh         kafka-console-consumer.sh  kafka-delete-records.sh      kafka-metadata-shell.sh              kafka-replica-verification.sh  kafka-streams-application-reset.sh  windows
kafka-acls.sh                 kafka-console-producer.sh  kafka-dump-log.sh            kafka-mirror-maker.sh                kafka-run-class.sh             kafka-topics.sh                     zookeeper-security-migration.sh
kafka-broker-api-versions.sh  kafka-consumer-groups.sh   kafka-features.sh            kafka-preferred-replica-election.sh  kafka-server-start.sh          kafka-verifiable-consumer.sh        zookeeper-server-start.sh
```
## Kafka Commands
The commands below use the Kafka binaries inside the Docker container, from the `bin` directory found above. This container includes Kafka 2.8.1 scripts, which use the `.sh` extension.

### Which broker address should I use?

- From a Kafka container on the Compose network, connect to `kafka:9092`.
- From Windows, Linux, or macOS host binaries, this Compose setup advertises `localhost:9094`. The current `docker-compose.yaml` publishes only port `9092`, so host binaries will need port `9094` published under the Kafka service before using `localhost:9094`.
- Keep host and container addresses distinct. A container should use the Compose service name; a host program should use a host-reachable address.

### Beginner workflow

#### 1. Check that the broker is reachable

Run this from the Kafka container's `bin` directory:

```sh
# Check the Kafka broker connection and supported API versions
./kafka-broker-api-versions.sh --bootstrap-server kafka:9092
```

#### 2. List existing topics

A topic is a named stream in which Kafka stores records.

```sh
# List topics in the Kafka cluster
./kafka-topics.sh --bootstrap-server kafka:9092 --list
```

#### 3. Create a topic for practice

This Compose setup has one Kafka broker, so use replication factor `1`. The topic has one partition for this introductory exercise.

```sh
# Create a one-partition topic on the single-broker cluster
./kafka-topics.sh --bootstrap-server kafka:9092 --create --topic beginner-demo --partitions 1 --replication-factor 1
```

If the topic already exists, Kafka reports that instead of creating a duplicate. Continue using the existing topic or choose another name.

#### 4. Inspect the topic

```sh
# Show the topic's partition count, replication factor, leader, and replicas
./kafka-topics.sh --bootstrap-server kafka:9092 --describe --topic beginner-demo
```

#### 5. Start a consumer in the first terminal

A consumer reads records from a topic. Run this in one terminal and leave it running. `--from-beginning` asks it to read existing records when this group has no saved offset yet.

```sh
# Read records from the practice topic as a named consumer group
./kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic beginner-demo --group beginner-demo-group --from-beginning
```

#### 6. Start a producer in a second terminal

Open a second PowerShell or terminal window and enter the Kafka container shell again. Navigate to the Kafka `bin` directory in that shell, then start the producer. Type one message per line; each line is sent as a separate record. Stop the producer with `Ctrl+C`.

```sh
# Start an interactive producer for the practice topic
./kafka-console-producer.sh --bootstrap-server kafka:9092 --topic beginner-demo
```

The consumer in the first terminal should display the records. Stop it with `Ctrl+C` when finished.

#### 7. Inspect the consumer group and its lag

A consumer group tracks the records its consumers have read. `CURRENT-OFFSET` is the group's saved position, `LOG-END-OFFSET` is the latest position in Kafka, and `LAG` is the difference between them.

```sh
# List consumer groups
./kafka-consumer-groups.sh --bootstrap-server kafka:9092 --list
```

```sh
# Show the practice group's offsets, members, and lag
./kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group beginner-demo-group
```

#### 8. Review commands before using them

Kafka tools display their supported options with `--help`. Read the help before using a command that changes topics, offsets, access controls, or broker settings.

```sh
# Display options supported by the topic-management tool
./kafka-topics.sh --help
```

### Common topic management

```sh
# Show topic details, including partitions and replicas
./kafka-topics.sh --bootstrap-server kafka:9092 --describe --topic beginner-demo
```

```sh
# Add partitions to the topic; Kafka cannot reduce a topic's partition count later
./kafka-topics.sh --bootstrap-server kafka:9092 --alter --topic beginner-demo --partitions 2
```

```sh
# Set a topic retention override to one day
./kafka-configs.sh --bootstrap-server kafka:9092 --entity-type topics --entity-name beginner-demo --alter --add-config retention.ms=86400000
```

```sh
# Display topic-level configuration overrides
./kafka-configs.sh --bootstrap-server kafka:9092 --entity-type topics --entity-name beginner-demo --describe
```

```sh
# Remove the retention override and return to the broker's default
./kafka-configs.sh --bootstrap-server kafka:9092 --entity-type topics --entity-name beginner-demo --alter --delete-config retention.ms
```

```sh
# Delete the practice topic and all records stored in it
./kafka-topics.sh --bootstrap-server kafka:9092 --delete --topic beginner-demo
```

Deleting a topic removes its data. Do not run the delete command for a topic whose records you need.

### Common consumer group operations

```sh
# List all consumer groups
./kafka-consumer-groups.sh --bootstrap-server kafka:9092 --list
```

```sh
# Show offsets, lag, and active members for a group
./kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group beginner-demo-group
```

```sh
# Show the group's active members and their partition assignments
./kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group beginner-demo-group --members --verbose
```

Offset reset operations change where a group resumes reading. Stop all consumers in that group first, review the proposed reset, and only then use the tool's `--execute` option. A reset can cause records to be read again or skipped.

### Kafka binary reference

These are the other tools shown in the Kafka 2.8.1 `bin` directory. Most are for cluster administration or specialized deployments; the beginner workflow above only needs the broker API, topic, producer, consumer, consumer-group, and config tools.

| Binary | Purpose | Notes for this Compose setup |
|---|---|---|
| `kafka-acls.sh` | View or change access-control lists | Requires Kafka authorization to be configured. |
| `kafka-broker-api-versions.sh` | Test broker connectivity and list supported APIs | Useful first connectivity check. |
| `kafka-cluster.sh` | Inspect or manage cluster metadata | Advanced administration. |
| `kafka-configs.sh` | Describe or change topic, broker, and other entity configurations | Changes can affect runtime behavior. |
| `kafka-console-consumer.sh` | Read topic records interactively | Used in the beginner workflow. |
| `kafka-console-producer.sh` | Write records interactively | Used in the beginner workflow. |
| `kafka-consumer-groups.sh` | Inspect groups, offsets, lag, and membership; reset offsets | Used in the beginner workflow; offset resets affect what a group reads. |
| `kafka-consumer-perf-test.sh` | Measure consumer throughput | Performance testing; can create load. |
| `kafka-delegation-tokens.sh` | Manage delegation tokens | Requires a configured security setup. |
| `kafka-delete-records.sh` | Delete records before specified offsets | Destructive; use only after reviewing the target offsets. |
| `kafka-dump-log.sh` | Inspect Kafka log-segment files | Advanced local storage troubleshooting. |
| `kafka-features.sh` | Inspect or change supported broker features | Advanced cluster administration. |
| `kafka-leader-election.sh` | Trigger partition leader elections | Advanced operations. |
| `kafka-log-dirs.sh` | Inspect broker log-directory and partition sizes | Broker diagnostics. |
| `kafka-metadata-shell.sh` | Inspect Kafka metadata files | Advanced troubleshooting; not part of the basic workflow. |
| `kafka-mirror-maker.sh` | Copy data between Kafka clusters | Requires a second cluster and configuration. |
| `kafka-preferred-replica-election.sh` | Restore preferred replica leadership | Cluster administration; limited value in a single-broker cluster. |
| `kafka-producer-perf-test.sh` | Measure producer throughput | Performance testing; can create load. |
| `kafka-reassign-partitions.sh` | Move partition replicas between brokers | Requires a reassignment plan and multiple brokers for typical use. |
| `kafka-replica-verification.sh` | Verify replica consistency | Cluster administration; useful with replicated topics. |
| `kafka-run-class.sh` | Run an internal Kafka Java class | Advanced tool; prefer the dedicated scripts above. |
| `kafka-server-start.sh` | Start a Kafka broker process | Compose already starts the broker; do not start a second broker in this container. |
| `kafka-server-stop.sh` | Stop a Kafka broker process | Use Docker Compose to stop this containerized broker. |
| `kafka-storage.sh` | Format or inspect Kafka storage for KRaft | This Compose setup uses ZooKeeper; do not use it to manage this broker. |
| `kafka-streams-application-reset.sh` | Reset Kafka Streams application state and offsets | For Kafka Streams applications, not the console consumer. |
| `kafka-verifiable-consumer.sh` | Run a consumer for verification and testing | Testing and diagnostics. |
| `kafka-verifiable-producer.sh` | Run a producer for verification and testing | Testing and diagnostics. |
| `connect-distributed.sh` | Start Kafka Connect in distributed mode | Requires a Kafka Connect worker configuration and service. |
| `connect-mirror-maker.sh` | Run Kafka Connect MirrorMaker 2 | Requires a Connect configuration and another cluster. |
| `connect-standalone.sh` | Start Kafka Connect in standalone mode | Requires a Kafka Connect worker configuration and connector plugins. |
| `trogdor.sh` | Run Kafka's workload and fault-testing framework | Specialized testing tool. |
| `zookeeper-security-migration.sh` | Migrate ZooKeeper security settings | Advanced security administration. |
| `zookeeper-server-start.sh` | Start a ZooKeeper server | Compose already starts ZooKeeper; do not start another one in this container. |
| `zookeeper-server-stop.sh` | Stop a ZooKeeper server | Use Docker Compose to stop the ZooKeeper container. |
| `zookeeper-shell.sh` | Connect to a ZooKeeper server shell | Advanced troubleshooting; avoid changing Kafka metadata manually. |

The `windows` subdirectory contains Windows `.bat` versions of Kafka tools. In PowerShell, use the `.bat` name from that directory and the same Kafka options. For host tools, use the host-reachable broker address described above.

### References

- [Apache Kafka 2.8 Quick Start](https://kafka.apache.org/28/getting-started/quickstart/)
- [Apache Kafka 2.8 Basic Kafka Operations](https://kafka.apache.org/28/operations/basic-kafka-operations/)
