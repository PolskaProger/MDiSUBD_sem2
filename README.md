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
# Лабораторная работа №2

# Шаги по созданию таблиц и триггеров

## 1. Создание таблиц STUDENTS и GROUPS

```sql
CREATE TABLE STUDENTS (
    ID NUMBER PRIMARY KEY,
    NAME VARCHAR2 (50),
    GROUP_ID NUMBER,
    CONSTRAINT fk_group_id FOREIGN KEY (GROUP_ID) REFERENCES GROUPS (ID) ON DELETE CASCADE
);

CREATE TABLE GROUPS (
    ID NUMBER PRIMARY KEY,
    NAME VARCHAR2 (50),
    C_VAL NUMBER,
    CONSTRAINT unique_group_name UNIQUE (NAME)
);
```

## 2. Триггеры

### Триггер для обеспечения уникальности поля ID в таблице STUDENTS

```sql
CREATE OR REPLACE TRIGGER trg_unique_id
BEFORE INSERT ON STUDENTS
FOR EACH ROW
DECLARE
    id_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO id_count FROM STUDENTS WHERE ID = :NEW.ID;
    IF id_count > 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'ID must be unique in the STUDENTS table');
    END IF;
END;
/
```

### Триггер для автоматического увеличения ID в таблице STUDENTS

```sql
CREATE SEQUENCE students_seq START WITH 1 INCREMENT BY 1;

CREATE OR REPLACE TRIGGER trg_autoincrement_id
BEFORE INSERT ON STUDENTS
FOR EACH ROW
BEGIN
    IF :NEW.ID IS NULL THEN
        SELECT students_seq.NEXTVAL INTO :NEW.ID FROM DUAL;
    END IF;
END;
/
```

### Триггер для обеспечения уникальности поля NAME в таблице GROUPS

```sql
CREATE OR REPLACE TRIGGER trg_unique_group_name
BEFORE INSERT ON GROUPS
FOR EACH ROW
DECLARE
    name_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO name_count FROM GROUPS WHERE NAME = :NEW.NAME;
    IF name_count > 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Group name must be unique in the GROUPS table');
    END IF;
END;
/
```

### Триггер для реализации отношения внешнего ключа с каскадным удалением между таблицами STUDENTS и GROUPS

```sql
CREATE OR REPLACE TRIGGER trg_cascade_delete
FOR DELETE ON GROUPS
COMPOUND TRIGGER
    TYPE ids_t IS TABLE OF NUMBER;
    ids ids_t := ids_t();

    BEFORE STATEMENT IS
    BEGIN
        ids.DELETE;
    END BEFORE STATEMENT;

    BEFORE EACH ROW IS
    BEGIN
        ids.EXTEND;
        ids(ids.COUNT) := :OLD.ID;
    END BEFORE EACH ROW;

    AFTER STATEMENT IS
    BEGIN
        FOR i IN 1..ids.COUNT LOOP
            DELETE FROM STUDENTS WHERE GROUP_ID = ids(i);
        END LOOP;
    END AFTER STATEMENT;
END trg_cascade_delete;
/
```

### Триггер для ведения журнала всех действий манипуляций с данными в таблице STUDENTS

```sql
CREATE TABLE STUDENTS_AUDIT (
    LOG_ID NUMBER PRIMARY KEY,
    OPERATION VARCHAR2 (50),
    STUDENT_ID NUMBER,
    STUDENT_NAME VARCHAR2 (100),
    GROUP_ID NUMBER,
    TIMESTAMP TIMESTAMP DEFAULT SYSTIMESTAMP
);

CREATE OR REPLACE TRIGGER trg_students_audit
BEFORE INSERT OR UPDATE OR DELETE ON STUDENTS
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO STUDENTS_AUDIT (LOG_ID, OPERATION, STUDENT_ID, STUDENT_NAME, GROUP_ID)
        VALUES ((SELECT NVL(MAX(LOG_ID), 0) + 1 FROM STUDENTS_AUDIT), 'INSERT', :NEW.ID, :NEW.NAME, :NEW.GROUP_ID);
    ELSIF UPDATING THEN
        INSERT INTO STUDENTS_AUDIT (LOG_ID, OPERATION, STUDENT_ID, STUDENT_NAME, GROUP_ID)
        VALUES ((SELECT NVL(MAX(LOG_ID), 0) + 1 FROM STUDENTS_AUDIT), 'UPDATE', :NEW.ID, :NEW.NAME, :NEW.GROUP_ID);
    ELSIF DELETING THEN
        INSERT INTO STUDENTS_AUDIT (LOG_ID, OPERATION, STUDENT_ID, STUDENT_NAME, GROUP_ID)
        VALUES ((SELECT NVL(MAX(LOG_ID), 0) + 1 FROM STUDENTS_AUDIT), 'DELETE', :OLD.ID, :OLD.NAME, :OLD.GROUP_ID);
    END IF;
END;
/
```

## 3. Хранимая процедура для восстановления данных с временным смещением

```sql
CREATE OR REPLACE PROCEDURE restore_data(
    p_date IN DATE DEFAULT NULL,
    p_offset IN NUMBER DEFAULT NULL
) IS
    v_target_date DATE;
BEGIN
    IF p_date IS NOT NULL THEN
        v_target_date := p_date;
    ELSIF p_offset IS NOT NULL THEN
        v_target_date := SYSDATE - (p_offset / 1440);
    ELSE
        RAISE_APPLICATION_ERROR(-20004, 'INVALID DATA');
    END IF;

    FOR rec IN (
        SELECT *
        FROM STUDENTS_AUDIT
        WHERE TIMESTAMP <= v_target_date
        ORDER BY TIMESTAMP DESC
    ) LOOP
        IF rec.OPERATION = 'INSERT' THEN
            DELETE FROM STUDENTS WHERE ID = rec.STUDENT_ID;
        ELSIF rec.OPERATION = 'UPDATE' THEN
            UPDATE STUDENTS
            SET NAME = rec.STUDENT_NAME, GROUP_ID = rec.GROUP_ID
            WHERE ID = rec.STUDENT_ID;
        ELSIF rec.OPERATION = 'DELETE' THEN
            INSERT INTO STUDENTS (ID, NAME, GROUP_ID)
            VALUES (rec.STUDENT_ID, rec.STUDENT_NAME, rec.GROUP_ID);
        END IF;
    END LOOP;
    COMMIT;
END;
/
```

## 4. Триггер для обновления столбца C_VAL в таблице GROUPS при изменении данных в таблице STUDENTS

```sql
CREATE OR REPLACE TRIGGER trg_update_c_val
FOR INSERT OR UPDATE OR DELETE ON STUDENTS
COMPOUND TRIGGER
    total_students NUMBER;

    BEFORE STATEMENT IS
    BEGIN
        total_students := 0;
    END BEFORE STATEMENT;

    AFTER EACH ROW IS
    BEGIN
        IF INSERTING THEN
            total_students := total_students + 1;
        ELSIF UPDATING THEN
            NULL;
        ELSIF DELETING THEN
            total_students := total_students - 1;
        END IF;
    END AFTER EACH ROW;

    AFTER STATEMENT IS
    BEGIN
        FOR r IN (SELECT GROUP_ID, COUNT(*) AS num_students FROM STUDENTS GROUP BY GROUP_ID) LOOP
            UPDATE GROUPS
            SET C_VAL = r.num_students
            WHERE ID = r.GROUP_ID;
        END LOOP;
    END AFTER STATEMENT;
END trg_update_c_val;
/
```

## 5. Тестирование таблиц

```sql
INSERT INTO GROUPS (ID, NAME, C_VAL) VALUES (1, 'Group A', 0);
INSERT INTO GROUPS (ID, NAME, C_VAL) VALUES (2, 'Group B', 0);

INSERT INTO STUDENTS (ID, NAME, GROUP_ID) VALUES (1, 'Student 1', 1);
INSERT INTO STUDENTS (ID, NAME, GROUP_ID) VALUES (2, 'Student 2', 1);
INSERT INTO STUDENTS (ID, NAME, GROUP_ID) VALUES (3, 'Student 3', 2);

SELECT * FROM GROUPS;
SELECT * FROM STUDENTS;
SELECT * FROM STUDENTS_AUDIT;
```
```
```
