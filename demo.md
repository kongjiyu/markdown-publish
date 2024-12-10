```sql
--Load table into heatwave cluster
CALL sys.heatwave_load(JSON_ARRAY('mysql_customer_orders'), NULL);

--Check whether it's loaded or not
USE performance_schema;
SELECT NAME, LOAD_STATUS FROM rpd_tables,rpd_table_id WHERE rpd_tables.ID = rpd_table_id.ID;

--Set Heatwave engine
SET SESSION use_secondary_engine=OFF;

--Compare Performance for select statement
select `o`.`ORDER_ID` AS `order_id`,`o`.`ORDER_DATETIME` AS `ORDER_DATETIME`,
    `o`.`ORDER_STATUS` AS `order_status`, `c`.`CUSTOMER_ID` AS `customer_id`,
    `c`.`EMAIL_ADDRESS` AS `email_address`,`c`.`FULL_NAME`  AS `full_name`,
    sum((`oi`.`QUANTITY` * `oi`.`UNIT_PRICE`)) AS `order_total`,
    `p`.`PRODUCT_NAME` AS `product_name`,`oi`.`LINE_ITEM_ID` AS `LINE_ITEM_ID`,
    `oi`.`QUANTITY`  AS `QUANTITY`,`oi`.`UNIT_PRICE` AS `UNIT_PRICE` 
from (((`orders` `o` join `order_items` `oi` on((`o`.`ORDER_ID` = `oi`.`ORDER_ID`))) 
    join `customers` `c` on((`o`.`CUSTOMER_ID` = `c`.`CUSTOMER_ID`))) 
    join `products` `p` on((`oi`.`PRODUCT_ID` = `p`.`PRODUCT_ID`))) 
group by `o`.`ORDER_ID`,`o`.`ORDER_DATETIME`,`o`.`ORDER_STATUS`,`c`.`CUSTOMER_ID`
    ,`c`.`EMAIL_ADDRESS` ,`c`.`FULL_NAME`,`p`.`PRODUCT_NAME`
    ,`oi`.`LINE_ITEM_ID`,`oi`.`QUANTITY`,`oi`.`UNIT_PRICE` limit 10;

--Dry Load Table in lakehouse
SET @db_list = '["mysql_customer_orders"]';
SET @dl_tables = '[{
"db_name": "mysql_customer_orders",
"tables": [{
    "table_name": "delivery_orders",
    "dialect": {
        "format": "csv",
        "field_delimiter": "\\t",
        "record_delimiter": "\\r\\n",
        "has_header": true,
        "is_strict_mode": false},
        "file": [{"par": "https://objectstorage.ap-singapore-1.oraclecloud.com/p/rt3jH0L8c4v980X13EJwhmFpoOhKQPCRLD8vFqo_PL4MDfBS7aNEut3QFW6ImDIM/n/nrdzgwhlhzo0/b/lakehouse-files/o/"}]
    }
]}
]';
SET @options = JSON_OBJECT('mode', 'dryrun', 'policy', 'disable_unsupported_columns', 'external_tables', CAST(@dl_tables AS JSON));
CALL sys.heatwave_load(@db_list, @options);

--Create table
SELECT log->>"$.sql" AS "Load Script" FROM sys.heatwave_autopilot_report WHERE type = "sql" ORDER BY id;

--Load Table
ALTER TABLE `mysql_customer_orders`.`delivery_orders` SECONDARY_LOAD;

--Join Table
select o.* ,d.*  from  orders o
join delivery_orders d on o.order_id = d.order_id
where o.order_id = 93751524;
```