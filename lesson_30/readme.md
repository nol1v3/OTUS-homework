# Урок 30 Nosql в Яндекс облаке 

## Практика

[практика](NoSQL_Yandex_cloud.md)

# Домашнее задание

Облака

## Цель

В результате выполнения ДЗ вы поработает с облаками.

Необходимо:

- одну из облачных БД заполнить данными (любыми из предыдущих дз);
- протестировать скорость запросов.

Задание повышенной сложности*

- сравнить 2-3 облачных NoSQL по скорости загрузки данных и времени выполнения запросов.

 
## Описание/Пошаговая инструкция выполнения домашнего задания:

[QuickStart](https://cloud.yandex.ru/ru/docs/managed-clickhouse/quickstart)

### Создание ВМ для клиентской части

В YC создвем ВМ с доступом извне и подключаемся к ней

Подключаем DEB-репозиторий

```bash
sudo apt-get install -y apt-transport-https ca-certificates dirmngr
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list
```

Устанавливаем зависимости и клиентское приложение clickhouse-client

```bash
sudo apt update && sudo apt install --yes clickhouse-client
```

Загружаем файл конфигурации для clickhouse-client

```bash
mkdir -p ~/.clickhouse-client && \
wget "https://storage.yandexcloud.net/doc-files/clickhouse-client.conf.example" \
  --output-document ~/.clickhouse-client/config.xml
```

### Создание кластера

[Дока](https://cloud.yandex.ru/ru/docs/managed-clickhouse/operations/cluster-create)

- В консоли управления выбираем каталог, в котором нужно создать кластер БД.
- Выбираем сервис Managed Service for ClickHouse.
- Нажимаем кнопку Создать кластер.
- Задаем параметры кластера и нажимаем кнопку Создать кластер.
- Дожидаемся, когда кластер будет готов к работе: его статус на панели Managed Service for ClickHouse® сменится на Running, а состояние — на Alive. Это может занять некоторое время.

![image](https://github.com/ada04/NoSQL/assets/40420948/fe431966-d032-4967-b97a-d08beef37085)


### Подключаемся к БД

Для подключения к серверу БД получаем SSL-сертификаты:

```bash
sudo mkdir --parents /usr/local/share/ca-certificates/Yandex/ && \
sudo wget "https://storage.yandexcloud.net/cloud-certs/RootCA.pem" \
     --output-document /usr/local/share/ca-certificates/Yandex/RootCA.crt && \
sudo wget "https://storage.yandexcloud.net/cloud-certs/IntermediateCA.pem" \
     --output-document /usr/local/share/ca-certificates/Yandex/IntermediateCA.crt && \
sudo chmod 655 \
     /usr/local/share/ca-certificates/Yandex/RootCA.crt \
     /usr/local/share/ca-certificates/Yandex/IntermediateCA.crt && \
sudo update-ca-certificates
```

Сертификаты будут сохранены в файлах:

    /usr/local/share/ca-certificates/Yandex/RootCA.crt
    /usr/local/share/ca-certificates/Yandex/IntermediateCA.crt

Используем для подключения ClickHouse® CLI, для этого укажем путь к SSL-сертификату RootCA.crt в конфигурационном файле, в элементе <caConfig>:

```
<config>
  <openSSL>
    <client>
      <loadDefaultCAFile>true</loadDefaultCAFile>
      <caConfig>/usr/local/share/ca-certificates/Yandex/RootCA.crt</caConfig>
      <cacheSessions>true</cacheSessions>
      <disableProtocols>sslv2,sslv3</disableProtocols>
      <preferServerCiphers>true</preferServerCiphers>
      <invalidCertificateHandler>
        <name>RejectCertificateHandler</name>
      </invalidCertificateHandler>
    </client>
  </openSSL>
</config>
```

Запустиv ClickHouse® CLI со следующими параметрами:

```bash
clickhouse-client --host rc1b-rlc7qkuh7cgoq7fp.mdb.yandexcloud.net --secure --user user1 --database db1 --port 9440 --password 4v6-n2C-3Gm-Fm3
```



![image](https://github.com/ada04/NoSQL/assets/40420948/b323aea5-87c4-4438-99af-266758668230)

### Выполнляем загрузку данных и запросы

[Взято из](https://github.com/ada04/NoSQL/edit/main/lesson09/readme.md)

![image](https://github.com/ada04/NoSQL/assets/40420948/caed9345-3553-4455-8b10-8cee08210c76)

#### Загружаем тестовый набор данных:

```bash
curl https://datasets.clickhouse.com/hits/tsv/hits_v1.tsv.xz | unxz --threads=`nproc` > hits_v1.tsv
curl https://datasets.clickhouse.com/visits/tsv/visits_v1.tsv.xz | unxz --threads=`nproc` > visits_v1.tsv
```

#### Создание таблиц

Таблицы будем создавать в БД **db1**

Мы создадим две таблицы:

- таблицу hits с действиями, осуществлёнными всеми пользователями на всех сайтах, обслуживаемых сервисом;
- таблицу visits, содержащую посещения — преднастроенные сессии вместо каждого действия.

Выполним операторы CREATE TABLE для создания этих таблиц:

```sql
CREATE TABLE db1.hits_v1
(
    `WatchID` UInt64,    `JavaEnable` UInt8,    `Title` String,    `GoodEvent` Int16,    `EventTime` DateTime,    `EventDate` Date,    `CounterID` UInt32,
    `ClientIP` UInt32,    `ClientIP6` FixedString(16),    `RegionID` UInt32,    `UserID` UInt64,    `CounterClass` Int8,    `OS` UInt8,    `UserAgent` UInt8,
    `URL` String,    `Referer` String,    `URLDomain` String,    `RefererDomain` String,    `Refresh` UInt8,    `IsRobot` UInt8,    `RefererCategories` Array(UInt16),
    `URLCategories` Array(UInt16),    `URLRegions` Array(UInt32),    `RefererRegions` Array(UInt32),    `ResolutionWidth` UInt16,    `ResolutionHeight` UInt16,
    `ResolutionDepth` UInt8,    `FlashMajor` UInt8,    `FlashMinor` UInt8,    `FlashMinor2` String,    `NetMajor` UInt8,    `NetMinor` UInt8,    `UserAgentMajor` UInt16,
    `UserAgentMinor` FixedString(2),    `CookieEnable` UInt8,    `JavascriptEnable` UInt8,    `IsMobile` UInt8,    `MobilePhone` UInt8,    `MobilePhoneModel` String,
    `Params` String,    `IPNetworkID` UInt32,    `TraficSourceID` Int8,    `SearchEngineID` UInt16,    `SearchPhrase` String,    `AdvEngineID` UInt8,
    `IsArtifical` UInt8,    `WindowClientWidth` UInt16,    `WindowClientHeight` UInt16,    `ClientTimeZone` Int16,    `ClientEventTime` DateTime,    `SilverlightVersion1` UInt8,
    `SilverlightVersion2` UInt8,    `SilverlightVersion3` UInt32,    `SilverlightVersion4` UInt16,    `PageCharset` String,    `CodeVersion` UInt32,    `IsLink` UInt8,
    `IsDownload` UInt8,    `IsNotBounce` UInt8,    `FUniqID` UInt64,    `HID` UInt32,    `IsOldCounter` UInt8,    `IsEvent` UInt8,    `IsParameter` UInt8,    `DontCountHits` UInt8,
    `WithHash` UInt8,    `HitColor` FixedString(1),    `UTCEventTime` DateTime,    `Age` UInt8,    `Sex` UInt8,    `Income` UInt8,    `Interests` UInt16,    `Robotness` UInt8,
    `GeneralInterests` Array(UInt16),    `RemoteIP` UInt32,    `RemoteIP6` FixedString(16),    `WindowName` Int32,    `OpenerName` Int32,    `HistoryLength` Int16,
    `BrowserLanguage` FixedString(2),    `BrowserCountry` FixedString(2),    `SocialNetwork` String,    `SocialAction` String,    `HTTPError` UInt16,    `SendTiming` Int32,
    `DNSTiming` Int32,    `ConnectTiming` Int32,    `ResponseStartTiming` Int32,    `ResponseEndTiming` Int32,    `FetchTiming` Int32,    `RedirectTiming` Int32,
    `DOMInteractiveTiming` Int32,    `DOMContentLoadedTiming` Int32,    `DOMCompleteTiming` Int32,    `LoadEventStartTiming` Int32,    `LoadEventEndTiming` Int32,
    `NSToDOMContentLoadedTiming` Int32,    `FirstPaintTiming` Int32,    `RedirectCount` Int8,    `SocialSourceNetworkID` UInt8,    `SocialSourcePage` String,    `ParamPrice` Int64,
    `ParamOrderID` String,    `ParamCurrency` FixedString(3),    `ParamCurrencyID` UInt16,    `GoalsReached` Array(UInt32),    `OpenstatServiceName` String,
    `OpenstatCampaignID` String,    `OpenstatAdID` String,    `OpenstatSourceID` String,    `UTMSource` String,    `UTMMedium` String,    `UTMCampaign` String,    `UTMContent` String,
    `UTMTerm` String,    `FromTag` String,    `HasGCLID` UInt8,    `RefererHash` UInt64,    `URLHash` UInt64,    `CLID` UInt32,    `YCLID` UInt64,    `ShareService` String,
    `ShareURL` String,    `ShareTitle` String,    `ParsedParams` Nested(Key1 String, Key2 String, Key3 String, Key4 String, Key5 String,  ValueDouble Float64),
    `IslandID` FixedString(16),    `RequestNum` UInt32,    `RequestTry` UInt8
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(EventDate)
ORDER BY (CounterID, EventDate, intHash32(UserID))
SAMPLE BY intHash32(UserID)
```

```sql
CREATE TABLE db1.visits_v1
(    `CounterID` UInt32,
    `StartDate` Date,
    `Sign` Int8,
    `IsNew` UInt8,
    `VisitID` UInt64,
    `UserID` UInt64,
    `StartTime` DateTime,
    `Duration` UInt32,
    `UTCStartTime` DateTime,
    `PageViews` Int32,
    `Hits` Int32,
    `IsBounce` UInt8,
    `Referer` String,
    `StartURL` String,
    `RefererDomain` String,
    `StartURLDomain` String,
    `EndURL` String,
    `LinkURL` String,
    `IsDownload` UInt8,
    `TraficSourceID` Int8,
    `SearchEngineID` UInt16,
    `SearchPhrase` String,
    `AdvEngineID` UInt8,
    `PlaceID` Int32,
    `RefererCategories` Array(UInt16),
    `URLCategories` Array(UInt16),
    `URLRegions` Array(UInt32),
    `RefererRegions` Array(UInt32),
    `IsYandex` UInt8,
    `GoalReachesDepth` Int32,
    `GoalReachesURL` Int32,
    `GoalReachesAny` Int32,
    `SocialSourceNetworkID` UInt8,
    `SocialSourcePage` String,
    `MobilePhoneModel` String,
    `ClientEventTime` DateTime,
    `RegionID` UInt32,
    `ClientIP` UInt32,
    `ClientIP6` FixedString(16),
    `RemoteIP` UInt32,
    `RemoteIP6` FixedString(16),
    `IPNetworkID` UInt32,
    `SilverlightVersion3` UInt32,
    `CodeVersion` UInt32,
    `ResolutionWidth` UInt16,
    `ResolutionHeight` UInt16,
    `UserAgentMajor` UInt16,
    `UserAgentMinor` UInt16,
    `WindowClientWidth` UInt16,
    `WindowClientHeight` UInt16,
    `SilverlightVersion2` UInt8,
    `SilverlightVersion4` UInt16,
    `FlashVersion3` UInt16,
    `FlashVersion4` UInt16,
    `ClientTimeZone` Int16,
    `OS` UInt8,
    `UserAgent` UInt8,
    `ResolutionDepth` UInt8,
    `FlashMajor` UInt8,
    `FlashMinor` UInt8,
    `NetMajor` UInt8,
    `NetMinor` UInt8,
    `MobilePhone` UInt8,
    `SilverlightVersion1` UInt8,
    `Age` UInt8,
    `Sex` UInt8,
    `Income` UInt8,
    `JavaEnable` UInt8,
    `CookieEnable` UInt8,
    `JavascriptEnable` UInt8,
    `IsMobile` UInt8,
    `BrowserLanguage` UInt16,
    `BrowserCountry` UInt16,
    `Interests` UInt16,
    `Robotness` UInt8,
    `GeneralInterests` Array(UInt16),
    `Params` Array(String),
    `Goals` Nested(
        ID UInt32,
        Serial UInt32,
        EventTime DateTime,
        Price Int64,
        OrderID String,
        CurrencyID UInt32),
    `WatchIDs` Array(UInt64),
    `ParamSumPrice` Int64,
    `ParamCurrency` FixedString(3),
    `ParamCurrencyID` UInt16,
    `ClickLogID` UInt64,
    `ClickEventID` Int32,
    `ClickGoodEvent` Int32,
    `ClickEventTime` DateTime,
    `ClickPriorityID` Int32,
    `ClickPhraseID` Int32,
    `ClickPageID` Int32,
    `ClickPlaceID` Int32,
    `ClickTypeID` Int32,
    `ClickResourceID` Int32,
    `ClickCost` UInt32,
    `ClickClientIP` UInt32,
    `ClickDomainID` UInt32,
    `ClickURL` String,
    `ClickAttempt` UInt8,
    `ClickOrderID` UInt32,
    `ClickBannerID` UInt32,
    `ClickMarketCategoryID` UInt32,
    `ClickMarketPP` UInt32,
    `ClickMarketCategoryName` String,
    `ClickMarketPPName` String,
    `ClickAWAPSCampaignName` String,
    `ClickPageName` String,
    `ClickTargetType` UInt16,
    `ClickTargetPhraseID` UInt64,
    `ClickContextType` UInt8,
    `ClickSelectType` Int8,
    `ClickOptions` String,
    `ClickGroupBannerID` Int32,
    `OpenstatServiceName` String,
    `OpenstatCampaignID` String,
    `OpenstatAdID` String,
    `OpenstatSourceID` String,
    `UTMSource` String,
    `UTMMedium` String,
    `UTMCampaign` String,
    `UTMContent` String,
    `UTMTerm` String,
    `FromTag` String,
    `HasGCLID` UInt8,
    `FirstVisit` DateTime,
    `PredLastVisit` Date,
    `LastVisit` Date,
    `TotalVisits` UInt32,
    `TraficSource` Nested(
        ID Int8,
        SearchEngineID UInt16,
        AdvEngineID UInt8,
        PlaceID UInt16,
        SocialSourceNetworkID UInt8,
        Domain String,
        SearchPhrase String,
        SocialSourcePage String),
    `Attendance` FixedString(16),
    `CLID` UInt32,
    `YCLID` UInt64,
    `NormalizedRefererHash` UInt64,
    `SearchPhraseHash` UInt64,
    `RefererDomainHash` UInt64,
    `NormalizedStartURLHash` UInt64,
    `StartURLDomainHash` UInt64,
    `NormalizedEndURLHash` UInt64,
    `TopLevelDomain` UInt64,
    `URLScheme` UInt64,
    `OpenstatServiceNameHash` UInt64,
    `OpenstatCampaignIDHash` UInt64,
    `OpenstatAdIDHash` UInt64,
    `OpenstatSourceIDHash` UInt64,
    `UTMSourceHash` UInt64,
    `UTMMediumHash` UInt64,
    `UTMCampaignHash` UInt64,
    `UTMContentHash` UInt64,
    `UTMTermHash` UInt64,
    `FromHash` UInt64,
    `WebVisorEnabled` UInt8,
    `WebVisorActivity` UInt32,
    `ParsedParams` Nested(
        Key1 String,
        Key2 String,
        Key3 String,
        Key4 String,
        Key5 String,
        ValueDouble Float64),
    `Market` Nested(
        Type UInt8,
        GoalID UInt32,
        OrderID String,
        OrderPrice Int64,
        PP UInt32,
        DirectPlaceID UInt32,
        DirectOrderID UInt32,
        DirectBannerID UInt32,
        GoodID String,
        GoodName String,
        GoodQuantity Int32,
        GoodPrice Int64),
    `IslandID` FixedString(16)
)
ENGINE = CollapsingMergeTree(Sign)
PARTITION BY toYYYYMM(StartDate)
ORDER BY (CounterID, StartDate, intHash32(UserID), VisitID)
SAMPLE BY intHash32(UserID)
```

Таблица hits_v1 использует базовый вариант движка MergeTree, а visits_v1 использует вариант Collapsing.

#### Импорт данных

Импорт данных в ClickHouse выполняется оператором INSERT INTO как в большинстве SQL-систем. Однако данные для вставки в таблицы ClickHouse обычно предоставляются в одном из поддерживаемых форматов вместо их непосредственного указания в предложении VALUES (хотя и этот способ поддерживается).

У нас загружены файлы в формате со значениями, разделёнными знаком табуляции; импортируем их, указав соответствующие запросы в аргументах командной строки:

```bash
clickhouse-client --query "INSERT INTO hits_v1 FORMAT TSV" --max_insert_block_size=100000 < hits_v1.tsv --user user1 --password 4v6-n2C-3Gm-Fm3 --database db1 --port 9440 --secure --host rc1b-rlc7qkuh7cgoq7fp.mdb.yandexcloud.net

clickhouse-client --query "INSERT INTO visits_v1 FORMAT TSV" --max_insert_block_size=100000 < visits_v1.tsv  --user user1 --password 4v6-n2C-3Gm-Fm3 --database db1 --port 9440 --secure --host rc1b-rlc7qkuh7cgoq7fp.mdb.yandexcloud.net
```

После импорта применим к таблицам оператор OPTIMIZE. 

```sql
OPTIMIZE TABLE hits_v1 FINAL
OPTIMIZE TABLE visits_v1 FINAL
```

Проверим, успешно ли загрузились данные:

```sql
SELECT COUNT(*) FROM hits_v1
SELECT COUNT(*) FROM visits_v1
```

![image](https://github.com/ada04/NoSQL/assets/40420948/111caa1b-57fb-485a-b403-643ea230a09d)

![image](https://github.com/ada04/NoSQL/assets/40420948/e74e1a6f-fcb5-416e-a8ce-a97af7edc3a7)

#### Примеры запросов

```sql
SELECT
    StartURL AS URL,
    AVG(Duration) AS AvgDuration
FROM visits_v1
WHERE StartDate BETWEEN '2014-03-20' AND '2014-03-22'
GROUP BY URL
ORDER BY AvgDuration DESC
LIMIT 10
```

![image](https://github.com/ada04/NoSQL/assets/40420948/f9b10d0e-c0d2-42c2-b884-ba8458487c58)


```sql
SELECT
    sum(Sign) AS visits,
    sumIf(Sign, has(Goals.ID, 1105530)) AS goal_visits,
    (100. * goal_visits) / visits AS goal_percent
FROM visits_v1
WHERE (CounterID = 912887) AND (toYYYYMM(StartDate) = 201403)
```

![image](https://github.com/ada04/NoSQL/assets/40420948/68efeabe-59c5-47cd-881b-95804c69accd)


### Сравнительный вывод
в ВМ на обычном ПК, время выполнения запросов составило 0.271 и 0.041 сек, в облаке 0.222 и 0.007 cек соответственно.

## Создание дополнительной тестовой БД (https://clickhouse.com/docs/en/getting-started/example-datasets) , протестировать скорость запросов

### Набор данных кулинарных рецептов

Я выбрал БД "Набор данных кулинарных рецептов".

Предварительно скачал файл, поэтому распаковываем и готовимся загружать.

```bash
unzip dataset.zip
```

#### Создадим БД и таблицу

```sql
CREATE TABLE recipes
(
    title String,
    ingredients Array(String),
    directions Array(String),
    link String,
    source LowCardinality(String),
    NER Array(String)
) ENGINE = MergeTree ORDER BY title
```

#### Добавим данные в таблицу

Чтобы добавить данные из файла full_dataset.csv в таблицу recipes, выполним команду:

```bash
clickhouse-client --query "
    INSERT INTO recipes
    SELECT
        title,
        JSONExtract(ingredients, 'Array(String)'),
        JSONExtract(directions, 'Array(String)'),
        link,
        source,
        JSONExtract(NER, 'Array(String)')
    FROM input('num UInt32, title String, ingredients String, directions String, link String, source LowCardinality(String), NER String')
    FORMAT CSVWithNames
" --input_format_with_names_use_header 0 --format_csv_allow_single_quote 0 --input_format_allow_errors_num 10 < dataset/full_dataset.csv --host rc1b-rlc7qkuh7cgoq7fp.mdb.yandexcloud.net --secure --user user1 --database db1 --port 9440 --password 4v6-n2C-3Gm-Fm3
```

**Пояснение:**

- набор данных представлен в формате CSV и требует некоторой предварительной обработки при вставке. Для предварительной обработки используется табличная функция input;
- структура CSV-файла задается в аргументе табличной функции input;
- поле num (номер строки) не нужно — оно считывается из файла, но игнорируется;
- при загрузке используется FORMAT CSVWithNames, но заголовок в CSV будет проигнорирован (параметром командной строки --input_format_with_names_use_header 0), поскольку заголовок не содержит имени первого поля;
- в файле CSV для разделения строк используются только двойные кавычки. Но некоторые строки не заключены в двойные кавычки, и чтобы одинарная кавычка не рассматривалась как заключающая, используется параметр --format_csv_allow_single_quote 0;
- некоторые строки из CSV не могут быть считаны корректно, поскольку они начинаются с символов\M/, тогда как в CSV начинаться с обратной косой черты могут только символы \N, которые распознаются как NULL в SQL. Поэтому используется параметр --input_format_allow_errors_num 10, разрешающий пропустить до десяти некорректных записей;
- массивы ingredients, directions и NER представлены в необычном виде: они сериализуются в строку формата JSON, а затем помещаются в CSV — тогда они могут считываться и обрабатываться как обычные строки (String). Чтобы преобразовать строку в массив, используется функция JSONExtract.

#### Проверим добавленние данных

Чтобы проверить добавленные данные, подсчитаем количество строк в таблице:

```sql
SELECT count()
FROM recipes
```

![image](https://github.com/ada04/NoSQL/assets/40420948/63f2ffcb-1493-4a21-b05f-1d265234619a)


#### Примеры запросов

Самые упоминаемые ингридиенты в рецептах:

```sql
SELECT
    arrayJoin(NER) AS ingridient,
    count() AS cnt
FROM recipes
GROUP BY ingridient
ORDER BY cnt DESC
LIMIT 10
```

![image](https://github.com/ada04/NoSQL/assets/40420948/649538c2-6fa0-4f45-9f94-e91f6efe740a)

10 rows in set. Elapsed: 0.514 sec. Processed 2.23 million rows, 361.57 MB (4.34 million rows/s., 703.21 MB/s.)
Peak memory usage: 32.91 MiB.


Самые сложные рецепты с яйцами

```sql
SELECT
    title,
    length(NER),
    length(directions)
FROM recipes
WHERE has(NER, 'eggs')
ORDER BY length(directions) DESC
LIMIT 5;
```

![image](https://github.com/ada04/NoSQL/assets/40420948/779d754d-e52c-4249-8bba-85e264698702)

5 rows in set. Elapsed: 1.435 sec. Processed 2.23 million rows, 1.65 GB (1.55 million rows/s., 1.15 GB/s.)
Peak memory usage: 30.95 MiB.


Существует свадебный торт, который требует целых 126 шагов для производства! Рассмотрим эти шаги:

```sql
SELECT arrayJoin(directions)
FROM recipes
WHERE title = 'Chocolate-Strawberry-Orange Wedding Cake';
```

126 rows in set. Elapsed: 0.016 sec. Processed 39.24 thousand rows, 7.80 MB (2.50 million rows/s., 497.48 MB/s.)
Peak memory usage: 11.98 MiB.

### Сравнительный вывод по 2 части

#### Время выполнения запросов на ВМ: 

10 rows in set. Elapsed: 0.215 sec. Processed 2.23 million rows, 1.48 GB (10.35 million rows/s., 6.86 GB/s.)
5 rows in set. Elapsed: 5.091 sec. Processed 2.23 million rows, 1.65 GB (438.21 thousand rows/s., 324.21 MB/s.)
126 rows in set. Elapsed: 0.016 sec. Processed 39.24 thousand rows, 7.80 MB (2.51 million rows/s., 499.22 MB/s.)

#### Время выполнения запросов в YC

10 rows in set. Elapsed: 0.514 sec. Processed 2.23 million rows, 361.57 MB (4.34 million rows/s., 703.21 MB/s.)
5 rows in set. Elapsed: 1.435 sec. Processed 2.23 million rows, 1.65 GB (1.55 million rows/s., 1.15 GB/s.)
126 rows in set. Elapsed: 0.016 sec. Processed 39.24 thousand rows, 7.80 MB (2.50 million rows/s., 497.48 MB/s.)

**Разница конечно есть, но на моих тестовых данных не очень заметна :)**

