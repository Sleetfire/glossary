## Из каких компонентов состоит кластер?

Кластер состоит из брокеров и контроллеров.

**Брокеры** - серверы, на которых запускается экземпляр Kafka в режиме "брокер". Такой сервер отвечает за хранение
данных (топиков, партиций и сообщений) и обработку запросов клиентов. Брокеры образуют собой ядро кластера Kafka. Их
количество варьируется в зависимости от профиля нагрузки и потребностей в репликации и шардировании.

**Контроллеры** - серверы, на которых запускается экземпляр Kafka в режиме "контроллер". Контроллеры хранят метаданные,
информацию о брокерах и синхронизируют их состояние. Обычно используется три контроллера на кластер Kafka, каждый из
которых находится в зоне доступности для повышения отказоустойчивости всего кластера.

---

## Что такое transaction log?

**Журнал транзакций (transaction log)** - это долговременная упорядоченная структура данных, причем, данные в такую
структуру можно только добавлять. Записи из этого журнала нельзя ни изменять, ни удалять. Информация считывается слева
направо; таким образом гарантируется правильный порядок элементов. (Данные добавляются с правой стороны).

Начальная позиция в логе называется log-start offset

Позиция сообщения, записанного последним - log-end offset

Позиция консумера в текущий момент времени - current offset

Расстояние между конечным оффсетом и текущим оффсетом называют лагом - это первое, за чем стоит следить в своих
приложениях.

> Внутри брокера есть папка logs, внутри которой есть папки с названием топика и номером партиции, например, А-0, А-1 и
> т.
> д. Внутри этих папок есть 3 файла с расширениями: .log, .index, .timeindex
> Файл с расширением .index - маппинг оффсета на позицию. Это работает так: берем оффсет и идем в файл .index и получаем
> оттуда позицию, с которой начинаем читать из .log файла (с какого байта).
> .timeindex - маппинг timestamp'а на оффсет.

_У лог-файла есть лимит, равный 1 гигабайту._

---

## Партиция. Что это такое? На что влияет? Кто есть лидер партиции?

**Партиция** - это фундаментальный механизм хранения данных в Kafka. Каждый топик в Kafka разделен на одну или несколько
партиций. Каждая партиция - это упорядоченная и неизменяемая последовательность сообщений, которая хранит данные. При
отправке сообщения в Kafka можно указать ключ и определить, в какую партицию отправить сообщение. Это позволяет
управлять упорядоченностью данных с одним ключом и предотвращать изменение порядка обработки.

Партиции используются, чтобы повысить пропускную способность и распределить нагрузку по всем брокерам в кластере.

Партиции делятся на сегменты. Количество сегментов у партиции может быть разным. Оно варьируется в зависимости от
интенсивности записи и настроек размера сегмента. Сегмент можно представить как лог-файл, работающий по принципу FIFO.

> У каждой партиции есть «лидер» (Leader) — брокер, который работает с клиентами. Именно лидер работает с продюсерами и
> в
> общем случае отдаёт сообщения консьюмерам. К лидеру осуществляют запросы фолловеры (Follower) — брокеры, которые
> хранят
> реплику всех данных партиций. Сообщения всегда отправляются лидеру и, в общем случае, читаются с лидера. Лидера
> партиции
> выбирают контроллеры.

---

## Какие политики очистки данных поддерживаются? Какая взаимосвязь с размером сегмента?

В контексте сегментов стоит вспомнить и об их ротации. Когда сегмент достигает своего предела, он закрывается и вместо
него открывается новый. Сегмент, в который сейчас записываются данные, называют активным сегментом - по сути это файл,
открытый процессом брокера. Закрытыми же называются те сегменты, в которых больше нет записи.

Закрытые сегменты удаляются по времени, размеру или ключу. Устаревание настраивается глобально или на топик. Размер
сегмента определяет как часто старые файлы будут сменять новые. Если вы пишете много больших сообщений, следует
увеличивать размер сегмента и, наоборот, не следует использовать большие сегменты, если вы редко пишете маленькие
сообщения.

---

## Что такое offset? Где хранится?

**Смещение (offset) в Kafka** — это индекс (порядковый номер), указывающий на положение записи в партиции топика.
Информация о смещениях (порядковые номера всех сообщений) хранится в специальном топике __consumer_offsets.

---

## Rebalance. Что это такое? В каких случаях возникает?

**Rebalance** - это процесс передачи партиции топика от одного (неактивного) подписчика к другому подписчику, который
является свободным (не принадлежит ни к одному из топиков) в данный момент времени. Подписчики считаются активными (
alive) до тех пор, пока они отправляют на сервер Kafka периодически контрольные сигналы (heartbeats), которые
представляют собой потоки сообщений, идущих в заданный разработчиком момент времени (timeout).

Если контрольный сигнал от подписчика не был получен сервером Kafka в заданный момент времени, Kafka автоматически
генерирует исключение (TimeOutException) и инициирует `перебалансировку`. Во время процесса перебалансировки подписчики
не
могут получать сообщения из топиков, что делает их недоступными. Кроме того, при перебалансировке подписчики утрачивают
свое текущее состояние. Например, если некоторые данные кэшировались (сохранялись в промежуточном буфере для быстрого
доступа к ним) подписчиком, то необходимо будет обновить все кэши, что значительно замедлит приложение на время
восстановления исходного состояния подписчика.

[Больше информации](https://kafka-school.ru/blogs/kafka-rebalance/)

---

## Какую функцию выполняет Zookeeper в кластере?

Kafka использует Zookeeper для достижения консенсуса состояния в распределённой системе: есть несколько вещей, с
которыми должен быть «согласен» каждый брокер и Zookeeper помогает достичь этого «согласия» внутри кластера.
Начиная с Kafka 3.4 необходимость в использовании Zookeeper отпала: для арбитража появился собственный протокол KRaft.
Он решает те же задачи, но на уровне брокеров.

**Zookeeper** — это выделенный кластер серверов для образования кворума-согласия и поддержки внутренних процессов Kafka.
Благодаря этому инструменту мы можем управлять кластером Kafka: добавлять пользователей и топики, задавать им настройки.
Также Zookeeper помогает обнаружить сбои контроллера, выбрать новый и сохранить работоспособность кластера Kafka. А ещё
он хранит в себе все авторизационные данные и ограничения или Access Control Lists при работе консумеров и продюсеров с
брокерами.

---

## Какие гарантии доставки предоставляет Kafka? Каким образом это достигается на каждом уровне?

acks - гарантия доставки:

- 0 (подтверждения доставки сообщений не нужны, возможны потери)
- 1 (подтверждение есть только от лидер-партиции, сообщения также могут потеряться, но только если лидер отправил ответ,
  что получил сообщения, после чего брокер упал)
- -1 (all) (продюсер ждет подтверждения отправки сообщений от всех ISR-реплик, включая лидера)

> В любых очередях есть выбор между скоростью доставки и расходами на надёжность.

- Семантика at-most once означает, что при доставке сообщений нас устраивают потери сообщений, но не их дубликаты. Это
  самая слабая гарантия, которую реализуют брокерами очередей.
- Семантика at-least once означает, что мы не хотим терять сообщения, но нас устраивают возможные дубликаты.
- Семантика exactly-once означает, что мы хотим доставить одно и только одно сообщение, ничего не теряя и ничего не
  дублируя. Но тогда нужно будет самому организовывать хранение оффсетов, так как встроенные способы не подойдут.
  Необходимо использовать транзакции.

Со стороны продюсера разработчик определяет надёжность доставки сообщения до Kafka с помощью параметра acks. Указывая 0
или none, продюсер будет отправлять сообщения в Kafka, не дожидаясь никаких подтверждений записи на диск со стороны
брокера. Это самая слабая гарантия. В случае выхода брокера из строя или сетевых проблем, вы так и не узнаете, попало
сообщение в лог или просто потерялось.
Указывая настройку в 1 или leader, продюсер при записи будет дожидаться ответа от брокера с лидерской партицией —
значит, сообщение сохранено на диск одного брокера. В этом случае вы получаете гарантию, что сообщение было получено по
крайней мере один раз, но это всё ещё не страхует вас от проблем в самом кластере.

Наконец, устанавливая acks в -1 или all, вы просите брокера с лидерской партицией отправить вам подтверждение только
тогда, когда запись попадёт на локальный диск брокера и в реплики-фолловеры. Число этих реплик устанавливает настройка
`min.insync.replicas`. Чтобы указать количество таких реплик, необходимо использовать настройку min.insync.replicas = 3

---

## Какие гарантии порядка предоставляет Kafka?

В Kafka порядок может быть гарантирован только в пределах раздела (партиции). Это означает, что если сообщения были
отправлены от продюсера в определенном порядке, брокер запишет их в партицию, и все потребители будут читать из нее
в том же порядке. Поэтому, естественно, в топике с одной партицией легче обеспечить упорядочивание по сравнению с
несколькими партициями. Однако с одной партицией трудно добиться параллелизма и балансировки нагрузки.

---

## Автоматическое удаление данных.

В Kafka поддерживается автоматическое удаление данных по TTL (time-to-live):

- удаляются целиком сегменты партиций (не отдельные сообщения).
- segment timestamp expired -> to delete.

---

## Отправка сообщений в Kafka.

1. Fetch metadata (в старых версиях через Zookeeper, в новых через спец. брокер), чтобы продюсер узнал из чего состоит
   кластер, где какие реплики находятся, какие из них являются лидерами. Эта операция выполняется синхронно. Если
   произойдет сбой, тогда операция зависнет на 60 секунд. Продюсер кэширует метаданные, а потом, если есть
   необходимость, обновляет их. Поэтому лучше всегда использовать одного продюсера.
2. Сериализация сообщения. Происходит сериализация ключи и значения. Например, с использованием StringSerializer.
3. Выбор партиции, куда пойдет сообщение. Опции:
    - explicit partition - указываем конкретную партицию
    - round-robin - просим кафку саму разобраться куда слать сообщения (по кругу, сначала в первую, потом по вторую и т.
      д.)
    - key-defined (key_hash % n) - партиция определяется по хэшу ключа.
4. Сжатие (компрессия) сообщения. Используется compression.codec
5. Собираем batch из нескольких сообщений для повышения производительности. Для этого используются настройки (batch.size
   и linger.ms).
6. Непосредственно отправка.

---

## Прием сообщений в Kafka.

1. Fetch metadata - получение состояние кластера, расположение топика, какие реплики являются лидерами и т. д.
2. Подключение к лидер-репликам всех партиций топика.

Несколько консьюмеров можно объединить в одну группу, которая называется Consumer Group, принадлежность к группе
определяется с помощью параметра <<group.id>>.

В кафке есть спец. топик, называемый __consumer_offsets, в котором хранятся оффсеты для каждой группы, каждой партиции,
например,
Partition A/0
Group X
Offset 2

> Он нужен для того, чтобы не было повторных чтений одних и тех же данных из Kafka.

Запись оффсета в __consumer_offsets называется `commit`. Они бывают:
* auto commit (at most once)
* manual commit (at least once)

Также есть настройка, отвечающая за длительность хранения __consumer_offsets: `offsets.retention.minutes` (7 дней). Это
значит, что если консьюмер-группа не подключалась к топику в течение 7 дней, тогда оффсет будет удален из __
consumer_offsets. Чтобы указать консьюмер группе, что делать дальше, нужно использовать настройку: `auto.offset.reset`.
Настройка принимает значения earliest (читает с первого оффсета) и latest (читает с последнего оффсета).

---

## Структура сообщения в Kafka

Key: "Alice"

Value: "Registered on your website"

Timestamp: "Jun. 25, 2020 at 2:06 p.m." (CreateTime, LogAppendTime)

Headers: [{"X-Generated-By": "web-host-12.eu-west2.slurm.io"}]

---

## Отличия Kafka от RabbitMQ

> RabbitMQ и Apache Kafka по-разному передают данные от производителей к потребителям. RabbitMQ — это брокер сообщений
> общего назначения, который отдает приоритет сквозной доставке сообщений. Kafka — это распределенная платформа для
> потоковой трансляции событий, поддерживающая непрерывный обмен большими данными в реальном времени.



