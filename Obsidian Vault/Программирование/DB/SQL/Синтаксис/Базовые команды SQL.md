
### Выборка все доступных таблиц
```sql
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public' AND table_type = 'BASE TABLE';
```


### Удаление таблиц
```sql
DROP TABLE IF EXISTS 'table_name' CASCADE;
```