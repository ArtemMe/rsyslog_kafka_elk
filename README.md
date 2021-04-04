# Рецепт для rsyslog kafka и elk

## Intro.
The stack is popular, but I have not found any suitable articles about connecting Kafka. You can found somewhere about Kafka somewhere about Logstash, but not altogether.

In this article, we will make a docker-compose file that will launch the entire system, and build an image that simulates an application with logs. We will also consider how you can check each system separately.

I don't want to create a detailed description of each application. It is just starting point for you to learn rsyslog and ELK

Don't forget, when adding a new service, you need to rebuild docker-compose:
```docker-compose build```

Github with full project: https://github.com/ArtemMe/rsyslog_kafka_elk

I split the project on release (tag). Every release is a new service like rsyslog or Kibana

## Rsyslog. (tag 0.1)
We will need two configs: one with the basic settings `/etc/rsyslog.conf`, the second` /etc/rsyslog.d/kafka-sender.conf` optional with settings for our needs

We will not delve into the rsyslog settings because you can dig into them for a long time. Just remember basic instructions: module, template, action
Let's take a look at an example of the file `/etc/rsyslog.d/kafka-sender.conf`:

```
# load module which use for sending message to kafka
module(load="omkafka") 

# Declare template for log with name "json_lines" :
template(name="json_lines" type="list" option.json="on") {  
        constant(value="{")
        constant(value="\"timestamp\":\"")      property(name="timereported" dateFormat="rfc3339")
        constant(value="\",\"message\":\"")     property(name="msg")
        constant(value="\",\"host\":\"")        property(name="hostname")
        constant(value="\",\"severity\":\"")    property(name="syslogseverity-text")
        constant(value="\",\"facility\":\"")    property(name="syslogfacility-text")
        constant(value="\",\"syslog-tag\":\"")  property(name="syslogtag")
        constant(value="\"}")
}

# Decalare action to send message to kafka broker in test_topic_1. Note how we use template json_lines and module omkafka
action(
        broker=["host.docker.internal:9092"]
        type="omkafka"
        template="json_lines"
        topic="test_topic_1"
        action.resumeRetryCount="-1"
        action.reportsuspension="on"
)
```
Remember topic name: test_topic_1

You can find the full list of property names there: https://www.rsyslog.com/doc/master/configuration/properties.html

Also note the main file `/ etc / rsyslog.conf` contains a line like:` $ IncludeConfig /etc/rsyslog.d / *. Conf`
This is a directive that tells us where else to read the settings for rsyslog. It is useful to separate common settings from specific ones

### Create an image for generating logs
The image will essentially just start rsyslog. In the future, we will be able to enter this container and generate logs.

You can find the Docker file in the `/rsyslog` folder. Let's look at the chunk of that file where on the first and second lines we copy our config. On the third line, we mount a folder for logs which will be generated

```
COPY rsyslog.conf /etc/
COPY rsyslog.d/*.conf /etc/rsyslog.d/

VOLUME ["/var/log"]
```

Building
```
docker build . -t rsyslog_kafka
```
Launch the container to check image.
```
docker run rsyslog_kafka
```

To check that we are writing logs, go to our container and call the command:
```
docker run --rm --network=rsyslog_kafka_elk_elk rsyslog_kafka bash -c `logger -p daemon.debug "This is a test."`
```

Let's look at folder  `/logs`:
You should find a string like this `This is a test.`.

Congratulations! You have configured rsyslog in your docker container!


## A bit about networking in docker containers.
Let's create our network in the docker-compose.yml file. In the future, each service can be launched to different machines. This is no problem.
```
networks:
  elk:
    driver: bridge
```

## Kafka (tag 0.2)
I took this repository as a basis: `https://github.com/wurstmeister/kafka-docker`
The resulting service is:
```
zookeeper:
  image: wurstmeister/zookeeper:latest
  ports:
    - "2181:2181"
  container_name: zookeeper
  networks:
    - elk

kafka:
  image: wurstmeister/kafka:0.11.0.1
  ports:
    - "9092:9092"
  environment:
    # The below only works for a macOS environment if you installed Docker for
    # Mac. If your Docker engine is using another platform/OS, please refer to
    # the relevant documentation with regards to finding the Host IP address
    # for your platform.
    KAFKA_ADVERTISED_HOST_NAME: docker.for.mac.localhost
    KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    KAFKA_CREATE_TOPICS: "logstash_logs:1:1"
  links:
    - zookeeper
  depends_on:
    - zookeeper
  container_name: kafka
  networks:
    - elk
```

We will be able to see what is in the Kafka topic when we launch our containers. First, you need to download Kafka. Here is a cool tutorial `https://kafka.apache.org/quickstart` but if it's short download here `https://www.apache.org/dyn/closer.cgi?path=/kafka/2.7.0/kafka_2.13-2.7.0.tgz `and unpack it to `/app` folder.
We need scripts in the `/bin` folder.

Now, we can connect to the container and execute a script to see if there are any entries inside the topic `test_topic_1` :
```
docker run --rm --network=rsyslog_kafka_elk_elk -v /app/kafka_2.13-2.7.0:/kafka wurstmeister/kafka:0.11.0.1 bash -c "/kafka/bin/kafka-console-consumer.sh --topic test_topic_1 --from-beginning --bootstrap-server 172.23.0.4:9092"
```

About the command itself: we connect to the rsyslog_kafka_elk_elk network, rsyslog_kafka_elk is the name of the folder where the docker-compose.yml file is located, and elk is the network that we specified. With the -v command, we mount scripts for Kafka into our container.

The result of command should be something like this:
```
{"timestamp":"2021-02-27T17:43:38.828970+00:00","message":" action 'action-1-omkafka' resumed (module 'omkafka') [v8.1901.0 try https://www.rsyslog.com/e/2359 ]","host":"c0dcee95ffd0","severity":"info","facility":"syslog","syslog-tag":"rsyslogd:"}
```

### Logstash (tag 0.3)

Configs are located in the `/ logstash` folder. `logstash.yml` - here we specify parameters for connecting to Elasticsearch

In the config, there is a setting for Kafka as for an incoming stream and a setting for elasticsearch as for an outgoing stream
```
input {
	beats {
		port => 5044
	}

	tcp {
		port => 5000
	}
    kafka
    {
        bootstrap_servers => "kafka:9092"
        topics => "test_topic_1"
    }
}

## Add your filters / logstash plugins configuration here

output {
	elasticsearch {
		hosts => "elasticsearch:9200"
		user => "elastic"
		password => "changeme"
		ecs_compatibility => disabled
	}
    
    file {

        path => "/var/logstash/logs/test.log"
        codec => line { format => "custom format: %{message}"}
    }
}
```
To monitor what goes into the Elastisearch and check the Logstesh is working properly, I  created a file output stream so logs will be written to test.log  file. The main thing is not to forget to add volume to docker-compose.yml
```
volumes:
  - type: bind
    source: ./logstash/config/logstash.yml
    target: /usr/share/logstash/config/logstash.yml
    read_only: true
  - type: bind
    source: ./logstash/pipeline
    target: /usr/share/logstash/pipeline
    read_only: true
  - ./logs:/var/logstash/logs
```

Check test.log file in your project. You should find logs from kafka

### Elasticsearch (tag 0.3)

This is the simplest configuration. We will launch the trial version, but you can turn on the open source one if you wish. Configs as usual in `/elasticsearch/config/`
```
## Default Elasticsearch configuration from Elasticsearch base image.
## https://github.com/elastic/elasticsearch/blob/master/distribution/docker/src/docker/config/elasticsearch.yml
#
cluster.name: "docker-cluster"
network.host: 0.0.0.0

## X-Pack settings
## see https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-xpack.html
#
xpack.license.self_generated.type: trial
xpack.security.enabled: true
xpack.monitoring.collection.enabled: true
```

Let's check the indexes of the elastic. Take as a basis a cool image of `praqma / network-multitool` and command curl: 
```
docker run --rm --network=rsyslog_kafka_elk_elk praqma/network-multitool bash -c "curl elasticsearch:9200/_cat/indices?s=store.size:desc -u elastic:changeme"
```
As the result of command:
```
The directory /usr/share/nginx/html is not mounted.
Over-writing the default index.html file with some useful information.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0green  open .monitoring-es-7-2021.02.28       QP1RL9ezRwmCFLe38dnlTg 1 0 1337 442   1.4mb   1.4mb
green  open .monitoring-es-7-2021.03.07       z0f-K-g7RhqDEbqnupfzPA 1 0  576 428   1.2mb   1.2mb
green  open .monitoring-logstash-7-2021.03.07 rKMYIZE9Q6mSR6_8SG5kUw 1 0  382   0 340.4kb 340.4kb
green  open .watches                          nthHo2KlRhe0HC-8MuT6rA 1 0    6  36 257.1kb 257.1kb
green  open .monitoring-logstash-7-2021.02.28 x98c3c14ToSqmBSOX8gmSg 1 0  363   0 230.1kb 230.1kb
green  open .monitoring-alerts-7              nbdSRkOSSGuLTGYv0z2L1Q 1 0    3   5  62.4kb  62.4kb
yellow open logstash-2021.03.07-000001        22YB7SzYR2a-BAgDEBY0bg 1 1   18   0  10.6kb  10.6kb
green  open .triggered_watches                sp7csXheQIiH7TGmY-EiIw 1 0    0  12   6.9kb   6.9kb
100   784  100   784    0     0  14254      0 --:--:-- --:--:-- --:--:-- 14254
```
We can see that the indices are being created and our elastic is alive. Let's connect Kibana now


### Kibana (tag 0.4)
This is what the service looks like

```
kibana:
  build:
    context: kibana/
    args:
      ELK_VERSION: $ELK_VERSION
  volumes:
    - type: bind
      source: ./kibana/config/kibana.yml
      target: /usr/share/kibana/config/kibana.yml
      read_only: true
  ports:
    - "5601:5601"
  networks:
    - elk
  depends_on:
    - elasticsearch
```
In the `/ kibana` folder we have a docker file to build an image and also settings for kibana:

```
server.name: kibana
server.host: 0.0.0.0
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
monitoring.ui.container.elasticsearch.enabled: true

## X-Pack security credentials
#
elasticsearch.username: elastic
elasticsearch.password: changeme
```

To enter the Kibana UI, you need to log in to the browser `localhost: 5601` (login/password is elasctic/changeme)
In the left menu, find Discover, click on it and create an index. I suggest this `logstash- *`

<img width="1253" alt="Create index pattern" src="https://user-images.githubusercontent.com/12798761/113514744-191f5f80-9579-11eb-8fb1-3fc9d22236b2.png">

