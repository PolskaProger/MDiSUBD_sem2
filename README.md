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

## Шаг 1: Создание временных таблиц

```sql
CREATE GLOBAL TEMPORARY TABLE TEMP_DELETED_GROUPS (
    GROUP_ID NUMBER,
    GROUP_NAME VARCHAR2(50)
) ON COMMIT DELETE ROWS;

CREATE GLOBAL TEMPORARY TABLE TEMP_GROUP_CHANGES (
    GROUP_ID NUMBER,
    OPERATION VARCHAR2(10)
) ON COMMIT DELETE ROWS;
```

## Шаг 2: Создание таблицы GROUPS

```sql
CREATE TABLE GROUPS (
    ID NUMBER NOT NULL ENABLE,
    NAME VARCHAR2(50) NOT NULL ENABLE,
    C_VAL NUMBER,
    CONSTRAINT PK_GROUPS PRIMARY KEY (ID)
);
```

## Шаг 3: Создание таблицы STUDENTS

```sql
CREATE TABLE STUDENTS (
    ID NUMBER NOT NULL ENABLE,
    NAME VARCHAR2(50) NOT NULL ENABLE,
    GROUP_ID NUMBER NOT NULL ENABLE,
    CONSTRAINT PK_STUDENTS PRIMARY KEY (ID)
);
```

## Шаг 4: Создание таблицы STUDENT_LOG

```sql
CREATE TABLE STUDENT_LOG (
    LOG_ID NUMBER PRIMARY KEY,
    ACTION_TYPE VARCHAR2(10),
    STUDENT_ID NUMBER,
    STUDENT_NAME VARCHAR2(50),
    GROUP_ID NUMBER,
    GROUP_NAME VARCHAR2(50),
    ACTION_TIME TIMESTAMP DEFAULT SYSTIMESTAMP
);
```

## Шаг 5: Создание последовательности для STUDENT_LOG

```sql
CREATE SEQUENCE STUDENT_LOG_SEQ
    START WITH 1
    INCREMENT BY 1
    NOCACHE
    NOCYCLE;
```

## Шаг 6: Триггер для автоинкремента ID в таблице GROUPS

```sql
CREATE OR REPLACE TRIGGER AUTO_INCREMENT_ID_GROUPS
BEFORE INSERT ON GROUPS
FOR EACH ROW
BEGIN
    IF :NEW.id IS NULL THEN
        SELECT NVL(MAX(id), 0) + 1 INTO :NEW.id FROM GROUPS;
    END IF;
END AUTO_INCREMENT_ID_GROUPS;
/
```

## Шаг 7: Триггер для автоинкремента ID в таблице STUDENTS

```sql
CREATE OR REPLACE TRIGGER AUTO_INCREMENT_ID_STUDENTS
BEFORE INSERT ON STUDENTS
FOR EACH ROW
BEGIN
    IF :NEW.id IS NULL THEN
        SELECT NVL(MAX(id), 0) + 1 INTO :NEW.id FROM STUDENTS;
    END IF;
END AUTO_INCREMENT_ID_STUDENTS;
/
```

## Шаг 8: Триггер для проверки уникальности имени группы

```sql
CREATE OR REPLACE TRIGGER CHECK_UNIQUE_NAME_GROUP
BEFORE INSERT OR UPDATE ON GROUPS
FOR EACH ROW
BEGIN
    DBMS_OUTPUT.PUT_LINE('Проверка группы: ID = ' || :NEW.ID || ', NAME = ' || :NEW.NAME);

    IF :NEW.NAME IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Имя группы не может быть NULL');
    END IF;

    IF :NEW.ID IS NULL THEN
        RAISE_APPLICATION_ERROR(-20003, 'ID группы не может быть NULL');
    END IF;

    -- Проверка уникальности имени группы
    FOR rec IN (SELECT 1 FROM GROUPS WHERE NAME = :NEW.NAME AND ID != :NEW.ID) LOOP
        RAISE_APPLICATION_ERROR(-20002, 'Имя группы должно быть уникальным');
    END LOOP;
END;
/
```

## Шаг 9: Триггер для журналирования удаления студента

```sql
CREATE OR REPLACE TRIGGER LOG_DELETE_STUDENT
BEFORE DELETE ON STUDENTS
FOR EACH ROW
DECLARE
    v_group_name VARCHAR2(50); -- Переменная для хранения имени группы
BEGIN
    -- Проверяем, была ли удалена группа для этого студента
    BEGIN
        -- Читаем имя группы из временной таблицы temp_deleted_groups_names
        SELECT t.group_name
        INTO v_group_name
        FROM temp_deleted_groups t
        WHERE t.group_id = :OLD.group_id;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
           -- Если имя группы нигде не найдено, записываем "UNKNOWN"
                    v_group_name := 'UNKNOWN GROUP' || :OLD.group_id;
    END;

    -- Вставка записи в STUDENT_LOG с автоинкрементом log_id и именем группы
    INSERT INTO STUDENT_LOG (log_id, action_type, student_id, student_name, group_id, group_name)
    VALUES (student_log_seq.NEXTVAL, 'DELETE', :OLD.id, :OLD.name, :OLD.group_id, v_group_name);

END;
/
```

## Шаг 10: Триггер для журналирования вставки студента

```sql
CREATE OR REPLACE TRIGGER LOG_INSERT_STUDENT
AFTER INSERT ON STUDENTS
FOR EACH ROW
DECLARE
    v_group_name VARCHAR2(100); -- Переменная для хранения имени группы
BEGIN
     -- Получаем имя группы из таблицы GROUPS
    SELECT name
    INTO v_group_name
    FROM GROUPS
    WHERE id = :NEW.group_id;

    -- Вставка записи в STUDENT_LOG с автоинкрементом log_id
    INSERT INTO STUDENT_LOG (log_id, action_type, student_id, student_name, group_id, group_name)
    VALUES (student_log_seq.NEXTVAL, 'INSERT', :NEW.id, :NEW.name, :NEW.group_id, v_group_name);
END;
/
```

## Шаг 11: Триггер для журналирования обновления студента

```sql
CREATE OR REPLACE TRIGGER LOG_UPDATE_STUDENT
AFTER UPDATE ON STUDENTS
FOR EACH ROW
DECLARE
    v_group_name VARCHAR2(100); -- Переменная для хранения имени группы
BEGIN
     -- Получаем имя группы из таблицы GROUPS
    SELECT name
    INTO v_group_name
    FROM GROUPS
    WHERE id = :OLD.group_id;
    -- Вставка записи в STUDENT_LOG с автоинкрементом log_id
    INSERT INTO STUDENT_LOG (log_id, action_type, student_id, student_name, group_id, group_name)
    VALUES (student_log_seq.NEXTVAL, 'UPDATE', :OLD.id, :OLD.name, :OLD.group_id, v_group_name);
END;
/
```

## Шаг 12: Триггер для проверки существования группы перед вставкой студента

```sql
CREATE OR REPLACE TRIGGER TRG_CHECK_GROUP_BEFORE_INSERT
BEFORE INSERT ON STUDENTS
FOR EACH ROW
DECLARE
    v_count NUMBER;
BEGIN

    SELECT COUNT(*) INTO v_count FROM GROUPS WHERE ID = :NEW.group_id;

    IF v_count = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Ошибка: Группа ' || :NEW.group_id || ' не существует.');
    END IF;
END;
/
```

## Шаг 13: Триггер для вставки удаленной группы во временную таблицу

```sql
CREATE OR REPLACE TRIGGER TRG_INSERT_DELETED_GROUP
BEFORE DELETE ON GROUPS
FOR EACH ROW
BEGIN
  DBMS_OUTPUT.PUT_LINE('trg_insert_deleted_group ');... -- Записываем ID удаленной группы в временную таблицу
  INSERT INTO temp_deleted_groups (group_id, group_name)
  VALUES (:OLD.id, :OLD.name);
END;
/
```

## Шаг 14: Триггер для удаления студентов при удалении группы (каскадное удаление)

```sql
CREATE OR REPLACE TRIGGER TRG_DELETE_STUDENTS
AFTER DELETE ON GROUPS
DECLARE
  CURSOR c IS
    -- Выбираем студентов, у которых group_id соответствует удаленной группе
    SELECT id FROM STUDENTS WHERE group_id IN (SELECT group_id FROM temp_deleted_groups);
BEGIN
  -- Удаляем всех студентов, чьи group_id совпадают с удаленной группой
  FOR r IN c LOOP
    DELETE FROM STUDENTS WHERE id = r.id;
  END LOOP;

  -- Очистка временной таблицы после удаления студентов

END;
/
```

## Шаг 15: Триггер для обновления счетчика студентов в группе

```sql
CREATE OR REPLACE TRIGGER TRG_RECALCULATE_GROUP_COUNT
AFTER INSERT OR UPDATE OR DELETE ON STUDENTS
DECLARE
  CURSOR c IS SELECT DISTINCT group_id FROM temp_group_changes;
  v_count NUMBER;
BEGIN
  -- Для каждой группы, которая изменилась, пересчитываем количество студентов
  FOR r IN c LOOP
    SELECT COUNT(*) INTO v_count
    FROM STUDENTS
    WHERE group_id = r.group_id;

    -- Обновляем количество студентов в таблице GROUPS
    UPDATE GROUPS
    SET c_val = v_count
    WHERE id = r.group_id;
  END LOOP;

  -- Очищаем временную таблицу после пересчета
  DELETE FROM temp_group_changes;
END;
/
```

## Шаг 16: Триггер для отслеживания изменений в таблице STUDENTS

```sql
CREATE OR REPLACE TRIGGER TRG_UPDATE_GROUP_COUNT
AFTER INSERT OR UPDATE OR DELETE ON STUDENTS
FOR EACH ROW
BEGIN
  -- Добавляем изменения в временную таблицу
  IF INSERTING OR UPDATING THEN
    INSERT INTO temp_group_changes (group_id, operation)
    VALUES (:NEW.group_id, 'INSERT');
    INSERT INTO temp_group_changes (group_id, operation)
    VALUES (:OLD.group_id, 'INSERT');
  ELSIF DELETING THEN
    INSERT INTO temp_group_changes (group_id, operation)
    VALUES (:OLD.group_id, 'DELETE');
  END IF;
END;
/
```

## Шаг 17: Процедура для восстановления данных студентов

```sql
CREATE OR REPLACE PROCEDURE restore_students_data(
    p_time IN TIMESTAMP,
    p_time_shift IN INTERVAL DAY TO SECOND DEFAULT INTERVAL '0' DAY
)
AS
    v_target_time TIMESTAMP;
    v_count NUMBER;  -- Объявляем переменную здесь
BEGIN
    DELETE FROM STUDENTS;

    -- Вычисляем целевое время для восстановления
    v_target_time := p_time + p_time_shift;

    DBMS_OUTPUT.PUT_LINE(v_target_time);

    -- Восстановление по времени
    FOR record IN (
        SELECT ACTION_TYPE, STUDENT_ID, STUDENT_NAME, GROUP_ID, ACTION_TIME, GROUP_NAME
        FROM STUDENT_LOG
        WHERE action_time <= v_target_time
        ORDER BY action_time
    ) LOOP
       IF record.action_type = 'INSERT' THEN
        -- Проверяем наличие группы с таким ID
        SELECT COUNT(*) INTO v_count
        FROM GROUPS
        WHERE id = record.group_id;

        -- Если группы нет, создаем новую
        IF v_count = 0 THEN
            INSERT INTO GROUPS (id, name, c_val)
            VALUES (record.group_id, record.group_name, 0);
        END IF;

        -- Вставляем студента
        INSERT INTO STUDENTS (id, name, group_id)
        VALUES (record.student_id, record.student_name, record.group_id);


        ELSIF record.action_type = 'DELETE' THEN
            DELETE FROM STUDENTS 
            WHERE id = record.student_id;

            -- -- Обновляем количество студентов в группе
            -- UPDATE GROUPS
            -- SET students_count = students_count - 1
            -- WHERE id = record.group_id;

        ELSE
         -- Проверяем наличие группы с таким ID
            SELECT COUNT(*) INTO v_count
            FROM GROUPS
            WHERE id = record.group_id;

            -- Если группы нет, создаем новую
            IF v_count = 0 THEN
                INSERT INTO GROUPS (id, name, c_val)
                VALUES (record.group_id, 'New Group ' || record.group_id, 0);
            END IF;

            UPDATE STUDENTS
            SET name = record.student_name,
                group_id = record.group_id
            WHERE id = record.student_id;
        END IF;
    END LOOP;

    COMMIT;
END restore_students_data;
/
```

## Шаг 18: SQL-коды для тестирования

1.  **Проверка автоинкремента:**

```sql
    INSERT INTO GROUPS (NAME, C_VAL) VALUES ('Group A', 10);
    INSERT INTO STUDENTS (NAME, GROUP_ID) VALUES ('Student 1', 1);
```

2.  **Проверка триггера уникальности имени группы:**

```sql
    INSERT INTO GROUPS (NAME, C_VAL) VALUES ('Group B', 5);
    -- Эта вставка должна вызвать ошибку
    INSERT INTO GROUPS (NAME, C_VAL) VALUES ('Group B', 8);
```

3.  **Проверка каскадного удаления:**

```sql
    INSERT INTO GROUPS (NAME, C_VAL) VALUES ('Group C', 12);
    INSERT INTO STUDENTS (NAME, GROUP_ID) VALUES ('Student 2', 3);
    INSERT INTO STUDENTS (NAME, GROUP_ID) VALUES ('Student 3', 3);

    DELETE FROM GROUPS WHERE ID = 3;
    -- Проверьте, что студенты Student 2 и Student 3 также удалены
```

4.  **Проверка журналирования:**

```sql
    INSERT INTO STUDENTS (NAME, GROUP_ID) VALUES ('Student 4', 1);
    UPDATE STUDENTS SET NAME = 'Updated Student 4' WHERE ID = 4;
    DELETE FROM STUDENTS WHERE ID = 4;

    SELECT * FROM STUDENT_LOG;
```

5.  **Проверка восстановления данных:**

```sql
    -- Вставляем данные
    INSERT INTO STUDENTS (NAME, GROUP_ID) VALUES ('Student 5', 2);
    COMMIT;
    -- Ждем немного
    -- Обновляем данные
    UPDATE STUDENTS SET NAME = 'Updated Student 5' WHERE ID = 5;
    COMMIT;
    -- Ждем немного
    -- Удаляем данные
    DELETE FROM STUDENTS WHERE ID = 5;
    COMMIT;

    -- Восстанавливаем данные на определенный момент времени
    DECLARE
        v_time TIMESTAMP := SYSTIMESTAMP - INTERVAL '1' MINUTE;
    BEGIN
        RESTORE_STUDENTS_DATA(v_time);
    END;
    /

    SELECT * FROM STUDENTS;
```

6.  **Проверка обновления счетчика студентов в группе:**

```sql
    INSERT INTO STUDENTS (NAME, GROUP_ID) VALUES ('Student 6', 1);
    -- Проверьте, что C_VAL для Group A увеличился
    UPDATE STUDENTS SET GROUP_ID = 2 WHERE ID = 6;
    -- Проверьте, что C_VAL для Group A уменьшился, а для Group B увеличился
    DELETE FROM STUDENTS WHERE ID = 6;
    -- Проверьте, что C_VAL для Group B уменьшился

    SELECT * FROM GROUPS;
```
