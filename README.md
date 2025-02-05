# MDiSUBD_sem2
## Лабораторная работа №1
### 1. Создание таблицы MyTable
```sql
CREATE TABLE MyTable (
    id NUMBER PRIMARY KEY,
    val NUMBER
);
```
### 2. Анонимный блок для вставки 10 000 случайных значений
```sql
BEGIN
  FOR i IN 1..10000 LOOP
    INSERT INTO MyTable (id, val)
    VALUES (i, ROUND(DBMS_RANDOM.VALUE(1, 1000)));
  END LOOP;
  COMMIT;
END;
/
```
### 3. Функция для проверки четности значений
```sql
CREATE OR REPLACE FUNCTION CheckEvenOdd RETURN VARCHAR2 IS
  even_count NUMBER;
  odd_count NUMBER;
BEGIN
  SELECT COUNT(CASE WHEN MOD(val, 2) = 0 THEN 1 END), 
         COUNT(CASE WHEN MOD(val, 2) != 0 THEN 1 END)
  INTO even_count, odd_count
  FROM MyTable;

  RETURN CASE
    WHEN even_count > odd_count THEN 'TRUE'
    WHEN even_count < odd_count THEN 'FALSE'
    ELSE 'EQUAL'
  END;
END;
/
```
Вызов функции
```sql
SELECT CheckEvenOdd() AS Result FROM DUAL;
```
### 4 Функция для генерации INSERT-команды
```sql
CREATE OR REPLACE FUNCTION GenerateInsertCommand(p_id NUMBER) RETURN VARCHAR2 IS
  v_val NUMBER;
  v_sql VARCHAR2(100);
BEGIN
  SELECT val INTO v_val
  FROM MyTable
  WHERE id = p_id;

  v_sql := 'INSERT INTO MyTable (id, val) VALUES (' || p_id || ', ' || v_val || ');';
  RETURN v_sql;

EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RETURN 'ID not found';
END;
/
```
Вызов функции
```sql
SELECT GenerateInsertCommand(5) FROM DUAL;
```

### 5. Процедуры для DML-операций
```sql
-- INSERT
CREATE OR REPLACE PROCEDURE InsertRow (
    p_id NUMBER,
    p_val NUMBER
) IS
BEGIN
    INSERT INTO MyTable (id, val) VALUES (p_id, p_val);
    COMMIT;
END;
/

-- UPDATE
CREATE OR REPLACE PROCEDURE UpdateRow (
    p_id NUMBER,
    p_new_val NUMBER
) IS
BEGIN
    UPDATE MyTable SET val = p_new_val WHERE id = p_id;
    COMMIT;
END;
/

-- DELETE
CREATE OR REPLACE PROCEDURE DeleteRow (
    p_id NUMBER
) IS
BEGIN
    DELETE FROM MyTable WHERE id = p_id;
    COMMIT;
END;
/
```
Вызов функции
```sql
EXEC InsertRow(10001, 777);
EXEC UpdateRow(10001, 888);
EXEC DeleteRow(10001);
```
### 6 Функция для расчета годового вознаграждения
```sql
CREATE OR REPLACE FUNCTION CalculateTotalBonus (
    p_salary NUMBER,
    p_bonus_percent NUMBER
) RETURN NUMBER IS
BEGIN
  IF p_salary <= 0 OR p_bonus_percent < 0 OR p_bonus_percent > 100 THEN
    RETURN NULL;
  END IF;

  RETURN (1 + p_bonus_percent / 100) * 12 * p_salary;

EXCEPTION
  WHEN OTHERS THEN
    RETURN NULL;
END;
/
```
Вызов функции
```sql
SELECT CalculateTotalBonus(5000, 10) FROM DUAL; -- 66000
```
