---
title: 常用sql
date: 2020-03-20
categories:
  - Tools
tags:
  - Tools
---
1. insert.. select
eg. insert into tb1 select * from tb2 ; 

2. create ... like
eg. create tb1 like tb2;

3. 查看所有库中包含指定字段的所有表
```
select * from information_schema where column_name='xxx';

-- 指定db
select * from information_schema where column_name='xxx' and table_schema='xxdb'; 

```

4. 批量修改表字段类型
```SQL
select CONCAT('alter table  ',TABLE_SCHEMA,'.',TABLE_NAME,'  
modify `', COLUMN_NAME, '` bigint(20) ',  
' default ', COLUMN_DEFAULT, ' COMMENT  '' , COLUMN_COMMENT, '';') 
from information_schema.COLUMNS 
where  COLUMN_NAME = 'first_order_id';
```
