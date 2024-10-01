# SQL - Best practices

## Avoid using `SELECT *`
- Only look for data you want to retrieve.

## Optimize `JOIN`
- Don't use `OUTER JOIN` unless necessary, it returns rows from **both** joining tables.

## Stored procedures
- Use for frequently used queries.

```sql
CREATE PROCEDURE popular_offers AS
BEGIN
...
END
```

## `EXISTS` over `IN`
Instead of
```sql
SELECT *
FROM employees
WHERE id IN (
  SELECT employee_id
  FROM employees_vacation
)
```
use
```sql
SELECT *
FROM employees
WHERE EXISTS (
  SELECT *
  FROM employees_vacation
  WHERE employees_vacation.employee_id = employees.id
)
```

## Indexes
- Increase query speed.
- Create on columns that will be often searched against.

```sql
CREATE INDEX my_index
ON employees (reference);
```

or for composite-columns.

```sql
CREATE INDEX my_composite_index
ON employees (first_name, last_name);
```