# Сравнение производительности баз данных ClickHouse и Postgres на основе баз данных Сокол

## Исходные данные

### Поток данных фотофиксации таблицы  ФотофиксацияТС

Структура таблицы  ФотофиксацияТС:
```
                    Таблица "public.ФотофиксацияТС"
        Столбец         |              Тип               | Модификаторы 
------------------------+--------------------------------+--------------
 primarykey             | uuid                           | NOT NULL
 createtime             | timestamp(3) without time zone | 
 creator                | character varying(255)         | 
 edittime               | timestamp(3) without time zone | 
 editor                 | character varying(255)         | 
 НомерТС                | character varying(20)          | 
 Скорость               | integer                        | 
 ОграничениеСкорости    | integer                        | 
 ИдентификаторМатериала | character varying(50)          | 
 Время                  | timestamp(3) without time zone | NOT NULL
 ДатаПоступления        | timestamp(3) without time zone | 
 Получатель             | character varying(255)         | 
 ТранспортноеСредство   | uuid                           | 
 Источник               | uuid                           | 
 x                      | numeric                        | 
 y                      | numeric                        | 
 ГруппаФакта            | uuid                           | 
 vehicletypetext        | character varying(100)         | 
 vehiclecolortext       | character varying(50)          | 
 vehiclebrandtext       | character varying(50)          | 
 vehiclemodeltext       | character varying(100)         | 
Индексы:
    "ФотофиксацияТС_origin_pkey" PRIMARY KEY, btree (primarykey)
    "ФотофиксацияТС_Время" btree ("Время") CLUSTER
    "ФотофиксацияТС_ГруппаФакта" btree ("ГруппаФакта")
    "ФотофиксацияТС_Источник" btree ("Источник")
    "ФотофиксацияТС_ИсточникВремя" btree ("Источник" NULLS FIRST, "Время" NULLS FIRST)
    "ФотофиксацияТС_НомерТСgin" gin ("НомерТС" gin_trgm_ops)
    "ФотофиксацияТС_НомерТСВремя" btree ("НомерТС", "Время")
    "ФотофиксацияТС_СкоростьВремя" btree ("Скорость", "Время") WHERE "Скорость" > 0
    "ФотофиксацияТС_ТранспСрво" btree ("ТранспортноеСредство") WHERE "ТранспортноеСредство" IS NOT NULL
    "ФотофиксацияТС_ТранспСрвоВремя" btree ("ТранспортноеСредство", "Время") WHERE "ТранспортноеСредство" IS NOT NULL
Ограничения внешнего ключа:
    "fk2694a00150f24e8499b49aaf61c5a1d5" FOREIGN KEY ("ГруппаФакта") REFERENCES "ГруппаФакта"(primarykey)
    "fka31d7a6b91b741fd930e793137af800c" FOREIGN KEY ("Источник") REFERENCES "Источник"(primarykey)
    "fkad4d18883bbb4fe7b3b58d1965df18e1" FOREIGN KEY ("ТранспортноеСредство") REFERENCES "ТранспортноеСредство"(primarykey)
Ссылки извне:
    TABLE ""СобытиеМониторинга"" CONSTRAINT "fkaf7894a5b6304b2eb8e0701436da676b" FOREIGN KEY ("ФотофиксацияТС") REFERENCES "ФотофиксацияТС"(primarykey)
Триггеры:
    insert_delete_fixation AFTER INSERT OR UPDATE ON "ФотофиксацияТС" FOR EACH ROW EXECUTE PROCEDURE insert_delete_fixation_func()

```
Число записей за 2018 год - `627 770 612`

### Поток данных фотофиксации таблицы  OdisseyEvents

Структура таблицы  OdisseyEvents
```
CREATE TABLE OdisseyEvents (
  uid UUID,
  typeid String,
  photo_id String,
  datetime DateTime,
  timestamp UInt32,
  object_id Int32,
  camera_direction_id Int32,
  camera_id Int32,
  grz String,
  speed UInt16
) ENGINE MergeTree() PARTITION BY toYYYYMM(datetime) ORDER BY (datetime, object_id, camera_direction_id, camera_id);
```

Число записей за 2018 год - `624 540 000`

## Тестовая среда

База данных Postges:
- Доступная оперативная память ~120Gb;

База данных lickHouse:
- Доступная оперативная память ~40Gb;

Процессоры идентичные - 32 ядра ~1200 MHz
Дисковые подсистемы аналогичные (ISCSI-диски)


## Сравнение производительности

Обе базы данных эффективно используют системые кэши и кэши базы данных.
Postgres за счет индексации данных и кэшировния результатов запросов 
показывает близкую (большую или меньшую) производительность при "повторяющихся" идентичных запросах.

Основное отличие в производительности возникет в ситуациях когда поступающие запросы 
(характерные для аналитики)
не позволяют базе данных использует кеши. 
В этом случае clickhouse  при объеме данных более XXX миллинов записей 
показыват свое преимущество.

### Запрос с использованием индекса

#### Postgres

Запрос:
```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2019-01-01 00:00:00' AND НомерТС='т459ар79';
```

Первый запрос:

0.00user 0.00system **0:00.02**elapsed **23%**CPU (0avgtext+0avgdata 6020maxresident)k

Второй запрос:

0.00user 0.00system **0:00.01**elapsed **27%**CPU (0avgtext+0avgdata 5988maxresident)k

#### СlickHouse

Запрос:
```
SELECT COUNT(*) FROM OdisseyEvents  
  WHERE datetime>'2018-01-01 00:00:00' AND  datetime<'2019-01-01 00:00:00' AND grz='т459ар79';
```
Первый запрос:

0.01user 0.01system **0:01.44**elapsed **2%**CPU (0avgtext+0avgdata 58456maxresident)k

Второй запрос:

0.01user 0.01system **0:01.37**elapsed **2%**CPU (0avgtext+0avgdata 52104maxresident)k


База | Первый запрос | Второй запрос
-----|---------------|--------------
Postgres | 0:00.02 | 0:00.01
ClickHouse | 0:01.44 | 0:01.37
||
Postgres/ClickHouse | 1/72 | 1/177

![Время выполнения запросов индексированных записей (секунд)](/images/index.png)


### Запрос без использованием индекса

#### Postgres

Запрос:
##### Один месяц:
```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2018-02-01 00:00:00' AND НомерТС LIKE 'р459%';
```

Результат при первом вызове (nocache):

0.00user 0.00system **0:06.89**elapsed **0%**CPU (0avgtext+0avgdata 5976maxresident)k

Результат при повторном вызове (использоваие cache):

0.01user 0.01system **0:04.07**elapsed **0%**CPU (0avgtext+0avgdata 52272maxresident)k


##### Квартал

```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2018-04-01 00:00:00' AND НомерТС LIKE 'а455%';
```


Результат при первом вызове (nocache):

00.00user 0.00system **2:45.01**elapsed **0%**CPU (0avgtext+0avgdata 6056maxresident)k

Результат при повторном вызове (использоваие cache):

0.01user 0.01system **0:04.60**elapsed **0%**CPU (0avgtext+0avgdata 52272maxresident)k

##### Год

```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2019-01-01 00:00:00' AND НомерТС LIKE 'в555%';
```

Результат при первом вызове (nocache):

0.00user 0.00system **6:02.29**elapsed **0%**CPU (0avgtext+0avgdata 5984maxresident)k

Результат при повторном вызове (использоваие cache):

0.01user 0.01system **0:05.10**elapsed **0%**CPU (0avgtext+0avgdata 52272maxresident)k


#### Clickhouse

##### Один месяц:

Запрос:

```
SELECT COUNT(*) FROM OdisseyEvents  
  WHERE datetime>'2018-01-01 00:00:00' AND  datetime<'2018-02-01 00:00:00' AND   match(grz, 'р459*');
```

Результат при первом вызове:

0.01user 0.01system **0:00.14**elapsed **20%**CPU (0avgtext+0avgdata 52272maxresident)k


##### Квартал

```
SELECT COUNT(*) FROM OdisseyEvents  
  WHERE datetime>'2018-01-01 00:00:00' AND  datetime<'2018-04-01 00:00:00' AND   match(grz, 'а455*');
```

Результат:

0.01user 0.01system **0:00.34**elapsed **10%**CPU (0avgtext+0avgdata 58624maxresident)k

##### Год

```
SELECT COUNT(*) FROM OdisseyEvents  
  WHERE datetime>'2018-01-01 00:00:00' AND  datetime<'2019-01-01 00:00:00' AND  match(grz, 'в555*');
```

Результат:

0.01user 0.01system **0:01.66**elapsed **1%**CPU (0avgtext+0avgdata 52220maxresident)k

База | Месяц(50 млн) | Квартал (150млн) | Год (600млн)
-----|---------------|------------------|------------
Postgres nocache | 0:06.89 | 2:45.01 | 6:02.29
Postgres cache| 0:04.07 |  0:04.60 | 0:05.10
ClickHouse | 0:00.14 | 0:00.34 | 0:01.66
||
Postgres nocache/ClickHouse | 50/1 | 485/1 | 217/1
Postgres cache/ClickHouse | 30/1 | 14/1 | 3/1

![Время выполнения запросов неиндексированных записей (секунд)](/images/noIndex.png)


### Кэширование

Вемя выполения повторных запрозов при "живом" кеше:

База | Месяц(50 млн) | Квартал (150млн) | Год (600млн)
-----|---------------|------------------|------------
Postgres | 0:04.07 |  0:04.60 | 0:05.10
ClickHouse | 0:00.14 | 0:00.34 | 0:01.66
||

### Подсчет числа записей

#### Postgres

Запрос:
```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2019-01-01 00:00:00';
```

Первый запрос:

0.00user 0.00system **1:13.45**elapsed **0%**CPU (0avgtext+0avgdata 5984maxresident)k

Второй запрос:

0.00user 0.00system **1:11.96**elapsed **0%**CPU (0avgtext+0avgdata 5940maxresident)k


#### СlickHouse

Запрос:
```
SELECT COUNT(*) FROM OdisseyEvents  
  WHERE datetime>'2018-01-01 00:00:00' AND  datetime<'2019-01-01 00:00:00';
```

Первый запрос:

0.01user 0.01system **0:00.35**elapsed **7%**CPU (0avgtext+0avgdata 54168maxresident)k

Второй запрос:

0.01user 0.01system **0:00.40**elapsed **7%**CPU (0avgtext+0avgdata 58620maxresident)k

База | Первый запрос | Второй запрос
-----|---------------|--------------
Postgres | 1:13.45 | 1:11.96
ClickHouse | 0:00.35 | 0:00.40


### Сравнение времени выполнения запросов на 600 млн записей (секунд)

База | С использованием индекса | Подсчет числа записей | Без использованием индекса
-----|--------------------------|-----------------------|---------------------------
Postgres | 0:00.02 | 1:13.45 | 6:02.29
ClickHouse | 0:01.44 | 0:00.35 | 0:01.66
||
Postgres/ClickHouse | 1/72 | 200/1 | 200/1

![Сравнение времени выполнения запросов на 600 млн записей (секунд)](/images/compare600.png)

## Портироваие Postgres-таблиц в Clickhouse

### DUMP таблиц из Postgres

#### DUMP таблицы ФотофиксацияТС


```
echo "COPY (select * from ФотофиксацияТС where Время>'2018-01-01' and Время>'2018-02-01') TO STDOUT;" | 
psql -U postgres -d БезопасныйГород >/var/data/tmp/ФотофиксацияТС.tsv
```

#### DUMP таблицы Источник

```
echo "COPY (select * from Источник) TO STDOUT;" | 
psql -U postgres -d БезопасныйГород >/var/data/tmp/Источник.tsv
```

#### DUMP таблицы Оборудование

```
echo "COPY (select * from Оборудование) TO STDOUT;" | 
psql -U postgres -d БезопасныйГород >/var/data/tmp/Оборудование.tsv
```

#### DUMP таблицы КомплексОборудования

```
echo "COPY (select * from КомплексОборудования) TO STDOUT;" | 
psql -U postgres -d БезопасныйГород >/var/data/tmp/КомплексОборудования.tsv
```

#### DUMP таблицы ФункциональныйМодуль

```
echo "COPY (select * from ФункциональныйМодуль) TO STDOUT;" | 
psql -U postgres -d БезопасныйГород >/var/data/tmp/ФункциональныйМодуль.tsv
```


### RESTORE таблиц в ClickHouse

### RESTORE таблицы ФотофиксацияТС

#### Создание таблицы ФотофиксацияТС


Все столбцы, которые могут иметь значение `NULL` описываются как `Nullable`.
```
CREATE TABLE `ФотофиксацияТС` (
  `primarykey` UUID, 
  `createtime` Nullable(DateTime), 
  `creator` String, 
  `edittime` Nullable(DateTime), 
  `editor` Nullable(String), 
  `НомерТС` String, 
  `Скорость` Int16, 
  `ОграничениеСкорости` Int16, 
  `ИдентификаторМатериала` String, 
  `Время` DateTime, 
  `ДатаПоступления` DateTime, 
  `Получатель` String, 
  `ТранспортноеСредство` Nullable(String), 
  `Источник` Nullable(UUID), 
  `x` Nullable(Float32), 
  `y` Nullable(Float32), 
  `ГруппаФакта` Nullable(String), 
  `vehicletypetext` Nullable(String), 
  `vehiclecolortext` Nullable(String), 
  `vehiclebrandtext` Nullable(String), 
  `vehiclemodeltext` Nullable(String)
  ) ENGINE = MergeTree() 
  PARTITION BY toYYYYMM(`Время`) 
  ORDER BY `Время` 
  SETTINGS index_granularity = 8192;
  ```

#### Формирование таблицы ФотофиксацияТС из DUMP'ов

Так как нетоторые столбцы типа `DateTime` могут содежать микосекунды при импорте данных
необходимо указать флаг `--date_time_input_format=best_effort`.
```
clickhouse-client -d БезопасныйГород \
  --date_time_input_format=best_effort \
  --query='INSERT INTO "ФотофиксацияТС"  \
  FORMAT TabSeparated' <var/data/tmp/ФотофиксацияТС.tsv
```

#### Размеры таблицы ФотофиксацияТС в ClickHouse

```
DUMP - 138 223 Mb
/var/lib/clickhouse/data/БезопасныйГород/ФотофиксацияТС - 24 717 Mb
```

### RESTORE таблицы Источник

#### Создание таблицы Источник

```
CREATE TABLE `БезопасныйГород`.`Источник` (
  `primarykey` UUID, 
  `createtime` Nullable(DateTime), 
  `creator` Nullable(String), 
  `edittime` Nullable(DateTime), 
  `editor` Nullable(String), 
  `Наименование` Nullable(String), 
  `Примечание` Nullable(String), 
  `НаправлениеУстановки` Nullable(String), 
  `ШаблонСсылкиНаМатериал` Nullable(String), 
  `СтрокаСоединения` Nullable(String), 
  `sipname` Nullable(String), 
  `ПолосаДвижения` Nullable(Int8), 
  `ДатаПоследнегоФакта` Nullable(DateTime), 
  `Идентификатор` Nullable(String), 
  `ПодвижныйОбъект` Nullable(UUID), 
  `МестоРасположения` Nullable(UUID), 
  `СтатусИсточника` Nullable(UUID), 
  `Оборудование` Nullable(UUID), 
  `КлючИсточника` Nullable(String), 
  `КолвоМест` Nullable(UInt16), 
  `ТипПарковки` Nullable(String), 
  `lat` Nullable(Float32), 
  `lon` Nullable(Float32)
  ) ENGINE = MergeTree() 
  PRIMARY KEY primarykey 
  ORDER BY primarykey 
  SETTINGS index_granularity = 8192;
```

#### Формирование таблицы Источник из DUMP'ов

```
clickhouse-client -d БезопасныйГород \
  --date_time_input_format=best_effort \
    --query='INSERT INTO "Источник"  \
      FORMAT TabSeparated'  <var/data/tmp/Источник.tsv
```

### RESTORE таблицы Оборудование

#### Создание таблицы Оборудование

```
CREATE TABLE `БезопасныйГород`.`Оборудование` (
  `primarykey` UUID, 
  `Идентификатор` String, 
  `Наименование` String, 
  `Актуально` UInt8, 
  `createtime` Nullable(DateTime), 
  `creator` String, 
  `edittime` Nullable(DateTime), 
  `editor` String, 
  `ФункциональныйМодуль` Nullable(UUID), 
  `ВидОборудования` Nullable(UUID), 
  `КомплексОборудования` Nullable(UUID)
  ) ENGINE = MergeTree() 
  PRIMARY KEY primarykey 
  ORDER BY primarykey 
  SETTINGS index_granularity = 8192;
```

### Формирование таблицы Оборудование из DUMP'а

Так как значения типа `boolean` в `ClickHouse` описываются как UInt8, до импорта необходимо преобразовать
раздклкнные табуляцией
- значения 't' в '1';
- значения 'f' в '0'.
```
sed -e "s/\tt\t/\t1\t" -e "\tf\t/\t0\t" <var/data/tmp/Оборудование.tsv |
clickhouse-client -d БезопасныйГород \
  --date_time_input_format=best_effort \
    --query='INSERT INTO "Оборудование"  \
      FORMAT TabSeparated' 
```

### RESTORE таблицы КомплексОборудования

#### Создание таблицы КомплексОборудования

```
CREATE TABLE `БезопасныйГород`.`КомплексОборудования` (
  `primarykey` UUID, 
  `Идентификатор` String, 
  `Наименование` String, 
  `Актуально` UInt8, 
  `createtime` Nullable(DateTime), 
  `creator` Nullable(String), 
  `edittime` Nullable(DateTime), 
  `editor` String, 
  `ФункциональныйМодуль` Nullable(UUID), 
  `place` String
  ) ENGINE = MergeTree() 
  PRIMARY KEY primarykey 
  ORDER BY primarykey 
  SETTINGS index_granularity = 8192;
```

### Формирование таблицы КомплексОборудования из DUMP'а

```
sed -e "s/\tt\t/\t1\t" -e "\tf\t/\t0\t" <var/data/tmp/КомплексОборудования.tsv |
clickhouse-client -d БезопасныйГород \
  --date_time_input_format=best_effort \
    --query='INSERT INTO "КомплексОборудования"  \
      FORMAT TabSeparated' 
```

### RESTORE таблицы ФункциональныйМодуль

#### Создание таблицы ФункциональныйМодуль

```
CREATE TABLE `БезопасныйГород`.`ФункциональныйМодуль` (
  `primarykey` UUID, 
  `Идентификатор` String, 
  `Наименование` String, 
  `Актуально` UInt8, 
  `createtime` Nullable(DateTime), 
  `creator` String, 
  `edittime` Nullable(DateTime), 
  `editor` String
  ) ENGINE = MergeTree() 
  PRIMARY KEY primarykey 
  ORDER BY primarykey 
  SETTINGS index_granularity = 8192;
```


### Формирование таблицы ФункциональныйМодуль из DUMP'а

```
sed -e "s/\tt\t/\t1\t" -e "\tf\t/\t0\t" <var/data/tmp/ФункциональныйМодуль.tsv |
clickhouse-client -d БезопасныйГород \
  --date_time_input_format=best_effort \
    --query='INSERT INTO "ФункциональныйМодуль"  \
      FORMAT TabSeparated' 
```

## Простые запросы 

### Postgres

Первый запрос:

```
echo "SELECT 
  НомерТС,
  Время 
FROM ФотофиксацияТС 
WHERE Время>'2018-01-01 00:00:00' AND  Время<'2019-01-01 00:00:00' AND НомерТС LIKE 'е622%';" |  
time psql -U postgres -d БезопасныйГород

0.37user 0.02system 5:22.96elapsed 0%CPU (0avgtext+0avgdata 14720maxresident)k
```

Второй запрос:

```
```



### ClickHouse

Первый запрос:

```
SELECT `НомерТС`,`Время` FROM `ФотофиксацияТС` WHERE `Время` >  '2018-01-01 00:00:00' AND  `Время` <'2019-01-01 00:00:00' AND  match(`НомерТС`, 'в555*') INTO OUTFILE '/tmp/req1';

SELECT 
    `НомерТС`, 
    `Время`
FROM `ФотофиксацияТС` 
WHERE (`Время` > '2018-01-01 00:00:00') AND (`Время` < '2019-01-01 00:00:00') AND match(`НомерТС`, 'в555*')
INTO OUTFILE '/tmp/req1'

846560 rows in set. Elapsed: 1.918 sec. Processed 627.77 million rows, 12.80 GB (327.36 million rows/s., 6.68 GB/s.) 

```

Второй запрос:

```
SELECT `НомерТС`,`Время` FROM `ФотофиксацияТС` WHERE `Время` >  '2018-01-01 00:00:00' AND  `Время` <'2019-01-01 00:00:00' AND  match(`НомерТС`, 'а621*') INTO OUTFILE '/tmp/req2';

SELECT 
    `НомерТС`, 
    `Время`
FROM `ФотофиксацияТС` 
WHERE (`Время` > '2018-01-01 00:00:00') AND (`Время` < '2019-01-01 00:00:00') AND match(`НомерТС`, 'а621*')
INTO OUTFILE '/tmp/req2'


585429 rows in set. Elapsed: 1.878 sec. Processed 627.77 million rows, 12.80 GB (334.28 million rows/s., 6.82 GB/s.)
```


## Запросы с операцией JOIN

### С WHERE на основной "большой" таблице

#### Postgres


#### ClickHouse

Первый запрос:

```
SELECT 
    `ФотофиксацияТС`.`НомерТС`, 
    `ФотофиксацияТС`.`Скорость`, 
    `Источник`.`Идентификатор` AS camera_id, 
    `Оборудование`.`Идентификатор` AS group_id, 
    `КомплексОборудования`.`Идентификатор` AS object_id
FROM 
(
    SELECT 
        `НомерТС`, 
        `Скорость`, 
        `Источник`
    FROM `ФотофиксацияТС` 
    WHERE match(`ФотофиксацияТС`.`НомерТС`, 'а621*')
) AS `ФотофиксацияТС` 
LEFT JOIN `Источник` ON `ФотофиксацияТС`.`Источник` = `Источник`.primarykey
LEFT JOIN `Оборудование` ON `Источник`.`Оборудование` = `Оборудование`.primarykey
LEFT JOIN `КомплексОборудования` ON `Оборудование`.`КомплексОборудования` = `КомплексОборудования`.primarykey
INTO OUTFILE '/tmp/reqjoinallc'

585429 rows in set. Elapsed: 3.185 sec. Processed 627.78 million rows, 12.81 GB (197.10 million rows/s., 4.02 GB/s.) 
```

Второй запрос:

```
select
:-]   `ФотофиксацияТС`.`НомерТС`,
:-]   `ФотофиксацияТС`.`Скорость`,
:-]   `Источник`.`Идентификатор` as camera_id,
:-]   `Оборудование`.`Идентификатор` as group_id,
:-]   `КомплексОборудования`.`Идентификатор` as object_id
:-] from (select `НомерТС`,`Скорость`,`Источник`  from `ФотофиксацияТС` where  match(`ФотофиксацияТС`.`НомерТС`, 'т355*')) as `ФотофиксацияТС`
:-] left join `Источник` on `ФотофиксацияТС`.`Источник`=`Источник`.`primarykey`
:-] left join `Оборудование` on `Источник`.`Оборудование`=`Оборудование`.`primarykey`
:-] left join `КомплексОборудования` on `Оборудование`.`КомплексОборудования`=`КомплексОборудования`.`primarykey`
:-] INTO OUTFILE '/tmp/reqjoinalld';

SELECT 
    `ФотофиксацияТС`.`НомерТС`, 
    `ФотофиксацияТС`.`Скорость`, 
    `Источник`.`Идентификатор` AS camera_id, 
    `Оборудование`.`Идентификатор` AS group_id, 
    `КомплексОборудования`.`Идентификатор` AS object_id
FROM 
(
    SELECT 
        `НомерТС`, 
        `Скорость`, 
        `Источник`
    FROM `ФотофиксацияТС` 
    WHERE match(`ФотофиксацияТС`.`НомерТС`, 'т355*')
) AS `ФотофиксацияТС` 
LEFT JOIN `Источник` ON `ФотофиксацияТС`.`Источник` = `Источник`.primarykey
LEFT JOIN `Оборудование` ON `Источник`.`Оборудование` = `Оборудование`.primarykey
LEFT JOIN `КомплексОборудования` ON `Оборудование`.`КомплексОборудования` = `КомплексОборудования`.primarykey
INTO OUTFILE '/tmp/reqjoinalld'


380348 rows in set. Elapsed: 3.189 sec. Processed 627.78 million rows, 12.81 GB (196.88 million rows/s., 4.02 GB/s.) 
```

## Запросы с операцией JOIN

### С WHERE на основной "малой" таблице

#### Postgres


#### ClickHouse

Запрос:
```
SELECT 
    `ФотофиксацияТС`.`НомерТС`, 
    `ФотофиксацияТС`.`Время`, 
    `ФотофиксацияТС`.`Скорость`, 
    `Источник`.`Идентификатор` AS camera_id, 
    `Оборудование`.`Идентификатор` AS group_id, 
    `КомплексОборудования`.`Идентификатор` AS object_id, 
    `КомплексОборудования`.`Наименование` AS place
FROM 
(
    SELECT 
        `КомплексОборудования`.primarykey, 
        `КомплексОборудования`.`Идентификатор`, 
        `КомплексОборудования`.`Наименование`
    FROM `КомплексОборудования` 
    WHERE match(`КомплексОборудования`.`Наименование`, '.*Ленина.*')
) AS `КомплексОборудования` 
LEFT JOIN 
(
    SELECT 
        `Оборудование`.primarykey, 
        `Оборудование`.`Идентификатор`, 
        `Оборудование`.`КомплексОборудования`
    FROM `Оборудование` 
) AS `Оборудование` ON `Оборудование`.`КомплексОборудования` = `КомплексОборудования`.primarykey
LEFT JOIN 
(
    SELECT 
        `Источник`.primarykey, 
        `Источник`.`Оборудование`, 
        `Источник`.`Идентификатор`
    FROM `Источник` 
) AS `Источник` ON `Источник`.`Оборудование` = `Оборудование`.primarykey
LEFT JOIN 
(
    SELECT 
        `НомерТС`, 
        `Время`, 
        `Скорость`, 
        `Источник`
    FROM `ФотофиксацияТС` 
) AS `ФотофиксацияТС` ON `ФотофиксацияТС`.`Источник` = `Источник`.primarykey

```


### С WHERE на JOIN-таблице

#### Postgres


#### ClickHouse


```
SELECT count(`ФотофиксацияТС`.`НомерТС`, `ФотофиксацияТС`.`Время`, `ФотофиксацияТС`.`Скорость`, `Источник`.`Идентификатор` AS camera_id, `Оборудование`.`Идентификатор` AS group_id, `КомплексОборудования`.`Идентификатор` AS object_id, `КомплексОборудования`.`Наименование` AS place)
FROM 
(
    SELECT 
        `НомерТС`, 
        `Время`, 
        `Скорость`, 
        `Источник`
    FROM `ФотофиксацияТС` 
) AS `ФотофиксацияТС` 
LEFT JOIN 
(
    SELECT 
        `Источник`.primarykey, 
        `Источник`.`Оборудование`, 
        `Источник`.`Идентификатор`
    FROM `Источник` 
) AS `Источник` ON `ФотофиксацияТС`.`Источник` = `Источник`.primarykey
LEFT JOIN 
(
    SELECT 
        `Оборудование`.primarykey, 
        `Оборудование`.`Идентификатор`, 
        `Оборудование`.`КомплексОборудования`
    FROM `Оборудование` 
) AS `Оборудование` ON `Источник`.`Оборудование` = `Оборудование`.primarykey
LEFT JOIN 
(
    SELECT 
        `КомплексОборудования`.primarykey, 
        `КомплексОборудования`.`Идентификатор`, 
        `КомплексОборудования`.`Наименование`
    FROM `КомплексОборудования` 
) AS `КомплексОборудования` ON `Оборудование`.`КомплексОборудования` = `КомплексОборудования`.primarykey
WHERE match(`КомплексОборудования`.`Наименование`, '.*Ленина.*')

┌─count(--ФотофиксацияТС.НомерТС, --ФотофиксацияТС.Время, --ФотофиксацияТС.Скорость, camera_id, group_id, Идентификатор, Наименование)─┐
│                                                                                                                             19700017 │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

1 rows in set. Elapsed: 166.007 sec. Processed 627.78 million rows, 27.24 GB (3.78 million rows/s., 164.07 MB/s.)
```

### Использование IN


#### ClickHouse
```
SELECT count(`НомерТС`, `Время`, `Скорость`, `Источник`) AS inLenina
FROM `ФотофиксацияТС` 
WHERE `Источник` IN 
(
    SELECT `Источник`.primarykey AS camera_uuid
    FROM 
    (
        SELECT 
            `КомплексОборудования`.primarykey, 
            `КомплексОборудования`.`Идентификатор`, 
            `КомплексОборудования`.`Наименование`
        FROM `КомплексОборудования` 
        WHERE match(`КомплексОборудования`.`Наименование`, '.*Ленина.*')
    ) AS `КомплексОборудования` 
    LEFT JOIN 
    (
        SELECT 
            `Оборудование`.primarykey, 
            `Оборудование`.`Идентификатор`, 
            `Оборудование`.`КомплексОборудования`
        FROM `Оборудование` 
    ) AS `Оборудование` ON `Оборудование`.`КомплексОборудования` = `КомплексОборудования`.primarykey
    LEFT JOIN 
    (
        SELECT 
            `Источник`.primarykey, 
            `Источник`.`Оборудование`, 
            `Источник`.`Идентификатор`
        FROM `Источник` 
    ) AS `Источник` ON `Источник`.`Оборудование` = `Оборудование`.primarykey
    WHERE `Источник`.primarykey != '00000000-0000-0000-0000-000000000000'
)

┌─count(НомерТС, Время, Скорость, Источник)─┐
│                                  31012371 │
└───────────────────────────────────────────┘

1 rows in set. Elapsed: 2.873 sec. Processed 627.78 million rows, 11.48 GB (218.52 million rows/s., 4.00 GB/s.) 
```


