# Quickstart

This exercise takes a file from kafka-connect `standalone` producer and sinks to kafka-connect `distributed`.
There will be a file called `standalone/connect-input-file/my-source-file.txt` note: you must leave an extra line at the bottom of the file for this to work.

-  Build the docker images as the connect image is required to build to include connector standalone configurations.

`docker-compose build`

-  Run the docker compose stack, it will run Zookeeper, Kafka and Kafka Connact Standalone and Kafka ConnactDistributed Connectors. This will scale the distributed connector to 3 running instances.

`docker-compose up --scale connect-distributed=3`

- Verify the docker images running in another terminal

`watch docker-compose ps`

- When running the docker-compose in some environments the folder where we will create the file from where the connector
is reading might be created as root user, this is due to how docker-compose and docker are setup and images are built,
change the folder permissions to your current user, in a terminal if necessary:

`sudo chown $USER: standalone/connect-input-file`

-  In a new terminal run the following. It will attach to the connect destination topic and wait for messages
for validation as this is a source connector.

`docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic simple-connect --from-beginning`

- A sample data file `my-source-file.txt` exists in `standalone/connect-input-file`  Open this file, add lines to it and notice that the lines are automatically read by
the connector and published to the kafka topic where we have attached our console client every time you save the file,
it relies in a new line / blank line at the end of the file, so make sure to add it in order for it to work properly.

- Retrieve the ports used by one of the distributed connector instances

```
docker ps
```

In my case, the ports were `0.0.0.0:32777`, `0.0.0.0:32778` and ``0.0.0.0:32779`


- Query the Connect REST API of the distributed connector to verify it is running

`curl http://localhost:32777/connectors`

- `cd distributed`

- Push this JSON file content to the REST API

`curl -XPUT -H "Content-Type: application/json"  --data "@connect-file-sink.json" http://localhost:32777/connectors/file-sink-connector/config | jq`

- If all goes well, you should see the following below

```yaml
    {
      "name": "file-sink-connector",
      "config": {
        "name": "file-sink-connector",
        "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
        "tasks.max": "1",
        "topics": "simple-connect",
        "file": "/tmp/my-output-file.txt",
        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "value.converter": "org.apache.kafka.connect.storage.StringConverter"
      },
      "tasks": [
        {
          "connector": "file-sink-connector",
          "task": 0
        }
      ],
      "type": "sink"
    }
```

- If data exists on the topic, then you should see data output to `distributed/connect-output-file/my-output-file.txt`.

- you can check the consumer groups via `kafka-consumer-groups`

`docker exec -it kafka /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --all-groups`

    GROUP                       TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                                   HOST            CLIENT-ID
    connect-file-sink-connector simple-connect  1          4667            4667            0               connector-consumer-file-sink-connector-1-f009d64b-b2ad-42e9-be14-a77b9acfc6c0 /172.23.0.6     connector-consumer-file-sink-connector-1
    connect-file-sink-connector simple-connect  0          4666            4666            0               connector-consumer-file-sink-connector-0-e413eb56-30ec-4a3b-89fc-bf4b2aea01a9 /172.23.0.5     connector-consumer-file-sink-connector-0
    connect-file-sink-connector simple-connect  2          4667            4667            0               connector-consumer-file-sink-connector-2-c34e154b-8efb-4b41-aea0-a133a5f8556c /172.23.0.7     connector-consumer-file-sink-connector-2



