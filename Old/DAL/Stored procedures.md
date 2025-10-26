Оголошення процедури:
POSTGRES:
```
CREATE PROCEDURE MyProc()
LANGUAGE SQL
BEGIN ATOMIC
SELECT * FROM qrtz_triggers;
END;
```
MS SQL:
```
CREATE PROCEDURE GETALLNOTES
AS
BEGIN
	SELECT * FROM Notes;
END;
```

З параметрами:
POSTGRES:
```
CREATE PROCEDURE insert_data(a integer, b integer)
LANGUAGE SQL
AS $$
INSERT INTO tbl VALUES (a);
INSERT INTO tbl VALUES (b);
$$;
```
MS SQL:
```
CREATE PROCEDURE GETALLNOTESWHERETITLECONTAINS(
				@titleContains VARCHAR(MAX) = NULL)
AS
BEGIN
	SELECT * FROM Notes
	WHERE Title LIKE '%' +  @titleContains +'%';
END;
```

Щоб оновити в MS SQL пишеш `ALTER`, а в POSTGRES `CREATE OR REPLACE`.
Щоб викликати процедуру в Postgres: ```
```
CALL GetAllUsers('parameter1value', 'parameter2value');
```
Щоб викликати процедуру в MS SQL: 
```
EXEC GetAllUsers @parameter1 = 'parameter1value'
```
**Нюанс**, в POSTGRES процедури не можуть повертати якісь дані, таблиці. Вони можуть тільки виконувати певні дії. Повертати дані можуть функції. 
Так оголошуються функції:
```
CREATE OR REPLACE FUNCTION GetAllUsers2()
RETURNS TABLE(id INT, name TEXT) 
LANGUAGE SQL
BEGIN ATOMIC
SELECT id, name  FROM Users;
END;
```
Виклик функції: `SELECT * FROM GetAllUsers2();`
