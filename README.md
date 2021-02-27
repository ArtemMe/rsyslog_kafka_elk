#RSYSLOG
Сервис rsyslog с конфигурацией /rsyslog/conf и дополнительными настройками в rsyslog.d/*
В данной конфигурации сохраняет логи в папку logs/*

#Kafka
Клевый гайд: https://kafka.apache.org/quickstart
Где скачать кафку:  https://www.apache.org/dyn/closer.cgi?path=/kafka/2.7.0/kafka_2.13-2.7.0.tgz 

Пример как можно выполнить и посмотреть что находится в топике кафки(образ можно взять вообще любой):

```
docker run --rm --network=rsyslog_kafka_elk_elk -v /Users/artemme/app/kafka_2.13-2.7.0:/kafka wurstmeister/kafka:0.11.0.1 bash -c "/kafka/bin/kafka-console-consumer.sh --topic test_topic_1 --from-beginning --bootstrap-server 172.23.0.4:9092"
```