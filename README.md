# kafka-->pubsub-->poc
| by Dr Christos Hadjinikoli |   
| Data Reply UK              |   
| 26/05/2017                 |   

## On the PubSub side
Before you begin make sure you have an active GCP account and that you have downloaded and installed the `google cloud SDK`.
1. Create project on Google Cloud Platform. An example projetc in this case is:
```Java
test-pubsub-project-final
```
2. By default, this project will have multiple service accounts associated with it (see "IAM & Admin" within GCP console). Within "IAM & Admin", find the tab for "Service Accounts". Click on "Create a new service account".
3. Provide a name  and click on the dropdown menu named "Role(s)". Under the "Pub/Sub" submenu, select "Pub/Sub Admin". Make sure to select `Furnish a new private key`. Doing this will create the service account and download a private key file to your local machine.
4. Finally, the key file that was downloaded to your machine needs to be placed on the machine running the framework. An environment variable named `GOOGLE_APPLICATION_CREDENTIALS` must point to this file. (**Tip**: export this environment variable as part of your shell startup file).

**Example download directory**: Place the key under `/kafka_2.11-0.10.2.1/gcc`, if the `gcc` is not present create it.
```Java
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/key/file
```
5. Next, authenticate the the gcp service account with:
```Java
gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
```
6. Go to your Google CloudPlatform--> API Manager and make sure that the following APIs are enabled--else enable them:
  * `Google Cloud Pub/Sub API`
  * `Google Cloud Resource Manager API`
7. Create a Pubsub topic: e.g. `my_pubsub_topic`.
8. Create a pull subscriber to the topic: e.g. `test_s`.
9. Select the new topic and choose "Publish Message" through the pubsub dashboard: e.g. ` Hello world`
10. Finally, use the following command on your local console to pull the message and test that everything works fine:
```Java
gcloud beta pubsub subscriptions pull test_s
```

### Troubleshooting GCLOUD Accounts
If you have multiple service accounts you may run into some problems, especially if you fail to set permissions properly. Make sure you use the correct service-account for your tests.

To see a list of the available accounts and set up `gcloud` to be linked to the correct one of them, type:
```Java
$ gcloud auth list      
> Credentialed Accounts:
> - owener.account@gmail.com
> - service-account-1@project-1.iam.gserviceaccount.com
> - service-account-2@project-2.iam.gserviceaccount.com ACTIVE
```
Notice that the currently active account with which all `gcloud` commands are associated with at this point is service-account-2. To set the active account to the one of your preference, run:
```Java
$ gcloud config set account `ACCOUNT`
```
Make sure you are also using the correct project to run this demo. To see a list of your active projects type in:
```Java
$ gcloud projects list      
> PROJECT_ID           NAME                 PROJECT_NUMBER
> my-new-project-1234  my-new-project-1234  12341234123412
> my-new-project-1234  my-new-project-1234  12341234123412
```
To set `gcloud` to reffer to the one of your preference, run:
```Java
$ gcloud config set project `PROJECT-NAME`
```

## On the `Kafka` side
For now, simply download the 0.10.2.0 release and un-tar it.
```Java
> tar -xzf kafka_2.11-0.10.2.0.tgz
> cd kafka_2.11-0.10.2.0
```
## On the kafka-pubsub-connector side
These instructions assume you are using `maven`.

1. Clone the repository, ensuring to do so recursively to pick up submodules:

```Java
git clone --recursive https://github.com/GoogleCloudPlatform/cloud-pubsub-kafka
```

2. Go to `cloud-pubsub-kafka/kafka-connector/` and make the jar that contains the connector:

```java
mvn clean package
```
The resulting jar is at `target/cps-kafka-connector.jar`.

3. Copy the  `/target/cps-kafka-connector.jar` into `kafka_2.11-0.10.2.1/libs`.
4. Go to `/cloud-pubsub-kafka/kafka-connecotr/config/` and copy the two example properties files.
5. Paste the files into `/kafka_2.11-0.10.2.1/config/`
6. For the purpose of this demo configure the `cps-sink-connector.properties` as follows:
```Java
name=CPSConnector
connector.class=com.google.pubsub.kafka.sink.CloudPubSubSinkConnector
tasks.max=10
topics=my_kafka_topic
cps.topic=my_pubsub_topic
cps.project=test-pubsub-project-final
```
7. Since we will be using string messages, in `connect-standalone.properties` file under `/kafka_2.11-0.10.2.1/config/`, replace:
```Java
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
```
with:
```Java
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.storage.StringConverter
```

## Putting it all together
1. Start `ZooKeeper`:
`Kafka` uses `ZooKeeper` so you need to first start a `ZooKeeper` server if you don't already have one. You can use the convenience script packaged with `kafka` to get a quick-and-dirty single-node `ZooKeeper` instance.
```Java
> bin/zookeeper-server-start.sh config/zookeeper.properties
[2013-04-22 15:01:37,495] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
...
```
2. Now start the `Kafka` server:
```Java
> bin/kafka-server-start.sh config/server.properties
[2013-04-22 15:01:47,028] INFO Verifying properties (kafka.utils.VerifiableProperties)
[2013-04-22 15:01:47,051] INFO Property socket.send.buffer.bytes is overridden to 1048576 (kafka.utils.VerifiableProperties)
...
```
3. Create a topic
Let's create a topic named "my_kafka_topic" with a single partition and only one replica:
```Java
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic my_kafka_topic
```
We can now see that topic if we run the list topic command:
```Java
> bin/kafka-topics.sh --list --zookeeper localhost:2181
my_kafka_topic
```
Alternatively, instead of manually creating topics you can also configure your brokers to auto-create topics when a non-existent topic is published to.

4. Send some messages
`Kafka` comes with a command line client that will take input from a file or from standard input and send it out as messages to the `Kafka` cluster. By default, each line will be sent as a separate message.

Run the producer and then type a few messages into the console to send to the server.
```Java
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my_kafka_topic
This is a message
This is another message
```
5. Start a consumer
`Kafka` also has a command line consumer that will dump out messages to standard output.
```Java
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my_kafka_topic --from-beginning
This is a message
This is another message
```
If you have each of the above commands running in a different terminal then you should now be able to type messages into the producer terminal and see them appear in the consumer terminal.

All of the command line tools have additional options; running the command with no arguments will display usage information documenting them in more detail.

6. Start a `Kafka`-->`PubSub` sink connector
Having set up your `pubsub` topic (in our case `my_pubsub_topic`) and your `Kafka` topic (in our case `my_kafka_topic`) go to `/kafka_2.11-0.10.2.1/` and start the connector by running th e following command:
```Java
bin/connect-standalone.sh config/connect-standalone.properties config/cps-sink-connector.properties
```
If this has worked fine, then all messages from the `my_kafka_topic` on `Kafka` producer must be passed to the `my_pubsub_topic` on `PubSub`.
7. To consume messages from `PubSub` rerun:
```Java
gcloud beta pubsub subscriptions pull test_s
```
