# Результаты и решение:

## 2. Создайте KSQL Stream WIKILANG
```
CREATE STREAM WIKILANG AS select 
	createdat 
	, channel 
	, username
	, wikipage
	, diffurl 
FROM WIKIPEDIANOBOT 
where isbot = false
and channel not like '%#en.%'
;

```
## 3. Мониторинг WIKILANG
```
describe extended wikilang;

```
![DESCRIBE_WIKILANG.JPG](https://github.com/adm-8/otus-de-andreevds-2019-11/blob/master/HW11_Lesson20/_images/DESCRIBE_WIKILANG.JPG?raw=true)

```
describe extended wikipedianobot;

```
![DESCRIBE_WIKIPEDIABOT.JPG](https://github.com/adm-8/otus-de-andreevds-2019-11/blob/master/HW11_Lesson20/_images/DESCRIBE_WIKIPEDIABOT.JPG?raw=true)

*Почему для wikipedianobot интерфейс показывает также consumer- метрики?*

**Вероятнее всего потому что наш стрим основан на WIKIPEDIANOBOT и является для него потребителем (consumer)**

## 4. Добавьте данные из стрима WIKILANG в ElasticSearch
*Используя полученные знания и документацию ответьте на вопросы:* 

*a) Опишите что делает каждая из этих операций?*
* set_elasticsearch_mapping_lang.sh - создает маппинг , т.е. описывает структуру данных
* submit_elastic_sink_lang_config.sh - создает соединение и использует созданный маппинг
* index name or pattern: wikilang - описывает имя индеса(ов) из которых брать данные

*б) Зачем Elasticsearch нужен mapping чтобы принять данные?*
* В маппинге могут быть описаны какие стринговые поля должны быть обработаны как "full text fields"
* какие поля содержат числа, даты или геолокацию
* формат дат
* кастомные правила обраотки для динамически добавляемых полей

*в) Что дает index-pattern?*
* index-pattern описывает имя индеса(ов) из которых брать данные


## 5. Создайте отчет "Топ10 национальных разделов" на базе индекса wikilang

*5.1 Что вы увидели в отчете?*

** В отчете мы видим кол-во записей с группировкой по CHANNEL, что-то вроде:**
```
select CHANNEL, count(*) from wikilang group by CHANNEL
```
![TOP10.JPG](https://github.com/adm-8/otus-de-andreevds-2019-11/blob/master/HW11_Lesson20/_images/TOP10.JPG?raw=true)


*5.2 Приложите тело запроса к заданию:*
```
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        },
        {
          "range": {
            "CREATEDAT": {
              "gte": 1583067441319,
              "lte": 1583068341319,
              "format": "epoch_millis"
            }
          }
        }
      ],
      "must_not": []
    }
  },
  "_source": {
    "excludes": []
  },
  "aggs": {
    "2": {
      "terms": {
        "field": "CHANNEL.keyword",
        "size": 10,
        "order": {
          "_count": "desc"
        }
      }
    }
  }
}
```

# 1. Разверните и подготовьте окружение

Вам потребуется компьютер или виртуальная машина минимум с двумя ядрами и 8 гигабайтами памяти (оптимально - 4 ядра, 16 гигабайт памяти и 100 гигабайт на диске для данных). Вы можете использовать Mac или Linux.
Вам потребуются такие пакеты:
- docker, docker-compose
- git
- jq
- OpenJDK8 (от него нужен keytool)
- curl

Если вы используете виртуальную машину с Linux рекомендую поднять значение vm.max_map_count (sudo sysctl -w vm.max_map_count=262144 для изменения на живой машине, и/или добавить строчку vm.max_map_count=262144 в /etc/sysctl.conf).
Если вы используете Mac - поднимите объем оперативной памяти для Docker Desktop до 8 гигабайт.

Скопируйте себе репозиторий с окружением:
`git clone https://github.com/confluentinc/cp-demo && cd cp-demo`

В эту же директорию скопируйте скрипты set_elasticsearch_mapping_lang.sh и submit_elastic_sink_lang_config.sh которые приложены к заданию (не забудьте сделать их исполняемыми командой `chmod u+x`).

Запустите скрипт `./scripts/start.sh` (вы можете остановить окружение и удалить все собранные данные если запустите `./scripts/stop.sh`)

В зависимости от мощности вашей машины и ширины канала в интернет скрипт может занимать от нескольких минут до получаса (повторые запуски будут проходить быстрее, так как все образы уже будут скачаны).

Если все прошло успешно, то скрипт поприветствует вас сообщением:
```
******************************************************************
DONE! Connect to Confluent Control Center at http://localhost:9021
******************************************************************
```

Если у вас появились проблемы - обратитесь к преподавателю, либо воспользуйтесь подсказками [на сайте Confluent](https://docs.confluent.io/current/tutorials/cp-demo/docs/index.html#troubleshooting-the-demo).

Для работы, помимо текстового редактора, вам понадобятся 4 окна:
1. Браузер (лучше Chrome) с Confluent Control Center http://localhost:9021 (если вы запускаетесь где-то на виртуалке - используйте ее адрес)
2. Браузер c Kibana UI http://localhost:5601
3. Терминал с открытым KSQL CLI, для этого в директории ./cp-demo выполните команду `docker-compose exec ksql-cli ksql http://ksql-server:8088`
4. Терминал с shell в директории ./cp-demo для запуска скриптов и отладки



# 2. Создайте KSQL Stream WIKILANG

Посмотрите какие топики есть сейчас в системе, и на основе того, в котором вы видите максимальный объем данных создайте stream по имени WIKILANG который фильтрует правки только в разделах национальных языков, кроме английского (поле channel вида #ru.wikipedia), который сделали не боты.

Stream должен содержать следующие поля: createdat, channel, username, wikipage, diffurl

# 3. Мониторинг WIKILANG

После 1-2 минут работы откройте Confluent Control Center и сравните пропускную способность топиков WIKILANG и WIKIPEDIANOBOT, какие числа вы видите?

- В KSQL CLI получите текущую статистику вашего стрима: describe extended wikilang;  

Приложите полный ответ на предыдущий запрос к ответу на задание.

- В KSQL CLI получите текущую статистику WIKIPEDIANOBOT: descrbie extended wikipedianobot;  

Приложите раздел Local runtime statistics к ответу на задание.  
Почему для wikipedianobot интерфейс показывает также consumer-* метрики?

# 4. Добавьте данные из стрима WIKILANG в ElasticSearch
- Добавьте mapping - запустите скрипт set_elasticsearch_mapping_lang.sh
- Добавьте Kafka Connect - запустите submit_elastic_sink_lang_config.sh
- Добавьте index-pattern - Kibana UI -> Management -> Index patterns -> Create Index Pattern -> Index name or pattern: wikilang -> кнопка Create

Используя полученные знания и документацию ответьте на вопросы:  
a) Опишите что делает каждая из этих операций?  
б) Зачем Elasticsearch нужен mapping чтобы принять данные?  
в) Что дает index-pattern?

# 5. Создайте отчет "Топ10 национальных разделов" на базе индекса wikilang
- Kibana UI -> Visualize -> + -> Data Table -> выберите индекс wikilang
- Select bucket type -> Split Rows, Aggregation -> Terms, Field -> CHANNEL.keyword, Size -> 10, нажмите кнопку Apply changes (выглядит как кнопка Play)
- Сохраните визуализацию под удобным для вас именем

Что вы увидели в отчете?

- Нажав маленьку круглую кнопку со стрелкой вверх под отчетом, вы сможете запросить не только таблицу, но и запрос на Query DSL которым он получен.

Приложите тело запроса к заданию.