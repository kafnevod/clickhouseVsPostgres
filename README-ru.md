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
показыват свое пеимущество.

### Подсчет числа записей

#### Postgres

Запрос:
```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2019-01-01 00:00:00';
```

Первый запрос:
```
count   
-----------
 620773085
(1 строка)

0.00user 0.00system 1:13.45elapsed 0%CPU (0avgtext+0avgdata 5984maxresident)k
0inputs+0outputs (0major+326minor)pagefaults 0swaps`
```

Второй запрос:
```
   count   
-----------
 620773085
(1 строка)

0.00user 0.00system 1:11.96elapsed 0%CPU (0avgtext+0avgdata 5940maxresident)k
0inputs+0outputs (0major+325minor)pagefaults 0swaps
```

#### СlickHouse

Запрос:
```
SELECT COUNT(*) FROM OdisseyEvents  
  WHERE datetime>'2018-01-01 00:00:00' AND  datetime<'2019-01-01 00:00:00';
```

Первый запрос:
```
624519723
0.01user 0.01system 0:00.35elapsed 7%CPU (0avgtext+0avgdata 54168maxresident)k
0inputs+0outputs (0major+2535minor)pagefaults 0swaps
```

Второй запрос:
```
624519723
0.01user 0.01system 0:00.40elapsed 7%CPU (0avgtext+0avgdata 58620maxresident)k
0inputs+0outputs (0major+1591minor)pagefaults 0swaps
```

### Запрос с использованием индекса

#### Postgres

Запрос:
```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2019-01-01 00:00:00' AND НомерТС='т459ар79';
```

Первый запрос:
```
 count 
-------
    11
(1 строка)

0.00user 0.00system 0:00.02elapsed 23%CPU (0avgtext+0avgdata 6020maxresident)k
0inputs+0outputs (0major+325minor)pagefaults 0swaps
```
Второй запрос:
```
count 
-------
    11
(1 строка)

0.00user 0.00system 0:00.01elapsed 27%CPU (0avgtext+0avgdata 5988maxresident)k
0inputs+0outputs (0major+327minor)pagefaults 0swaps
```

#### СlickHouse

Запрос:
```
SELECT COUNT(*) FROM OdisseyEvents  
  WHERE datetime>'2018-01-01 00:00:00' AND  datetime<'2019-01-01 00:00:00' AND grz='т459ар79';
```
Первый запрос:
```
11
0.01user 0.01system 0:01.44elapsed 2%CPU (0avgtext+0avgdata 58456maxresident)k
0inputs+0outputs (0major+1589minor)pagefaults 0swaps
```

Второй запрос:
```
11
0.01user 0.01system 0:01.37elapsed 2%CPU (0avgtext+0avgdata 52104maxresident)k
0inputs+0outputs (0major+3076minor)pagefaults 0swaps
```

### Запрос без использованием индекса

#### Postgres

Запрос:
##### Один месяц:
```
SELECT COUNT(*) FROM ФотофиксацияТС  
  WHERE Время>'2018-01-01 00:00:00' AND  Время<'2018-02-01 00:00:00' AND НомерТС LIKE 'р459%';
```

Результат:
```
 count 
-------
  1625
(1 строка)

0.00user 0.00system 0:06.89elapsed 0%CPU (0avgtext+0avgdata 5976maxresident)k
0inputs+0outputs (0major+329minor)pagefaults 0swaps
```

##### Квартал

```
