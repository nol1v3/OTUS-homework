## Создаем Kafka в docker.
```bash
docker pull apache/kafka:3.7.1
docker run -p 9092:9092 apache/kafka:3.7.1
```
## Взаимодействие с контейнером Kafka с локального компьютера.
kafkacat предлагает простой интерфейс командной строки для взаимодействия с Kafka. Это отличный инструмент для проверки работоспособности Kafka. 

Чтобы установить kafkacat, следуйте инструкциям на странице https://github.com/edenhill/kcat в зависимости от вашей операционной системы.

Чтобы проверить, работает ли Kafka, выполните следующую команду, чтобы получить список всех тем, которые сейчас есть в Kafka:
```bash
kcat -b localhost:9092 -L  # list all topics currently in kafka
Metadata for all topics (from broker 1: localhost:9093/1):
1 brokers:
broker 1 at localhost:9092 (controller)
0 topics:
```
Чтобы протестировать производителя, выполните следующую команду:
```bash
kcat -b localhost:9092 -t test-topic -P  # producer
one line per message
another line
```
Разделителем по умолчанию между сообщениями является новая строка. Когда вы закончите, нажмите ctrl-d, чтобы отправить сообщения. (К сведению: нажатие ctrl+c не сработает, вам придется повторить попытку.)

Чтобы прочитать созданные вами сообщения, выполните следующую команду, чтобы запустить потребителя:
```bash
kcat -b localhost:9092 -t test-topic -C  # consumer
one line per message
another line
% Reached end of topic test-topic [0] at offset 2
```

Однако публикация произвольного текста — это не совсем то, что нам нужно, поэтому давайте попробуем вместо этого публиковать сообщения JSON. Чтобы сделать это проще для расширения мы напишем несколько скриптов Python для создания и использования сообщений.

Для начала необходимо установить библиотеку kafka-python. Можно сделать это, запустив pip install kafka-python.
```python
# producer.py
from kafka import KafkaProducer
from datetime import datetime
import json
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)
producer.send('posts', {'author': 'choyiny', 'content': 'Kafka is cool!', 'created_at': datetime.now().isoformat()})
```

Ниже приведен пример потребителя Python, который подписывается на post и выводит каждое значение.
```python
# consumer.py
from kafka import KafkaConsumer
import json
consumer = KafkaConsumer(
    'posts',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)
# note that this for loop will block forever to wait for the next message
for message in consumer:
    print(message.value)
```

## Kafka *.sh
Сообщения можно отправлять с помощью скриптов kafka *.sh

kafka-topics.sh — создает топик, куда будем отправлять сообщение.
kafka-console-producer.sh — создает обращение издателя, который отправляет сообщение.
kafka-console-consumer.sh — формирует запрос к брокеру и получает сообщение.

```
/opt/kafka/bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic Test

echo "Hello, World from Kafka" | /opt/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic Test

/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic Test --from-beginning
```
