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




