
Users

| id  | Name    | isMaried | HiringDate | TechStack | YearsOfExp |
| --- | ------- | -------- | ---------- | --------- | ---------- |
| 1   | Jack    | true     | 19.06.2020 | .net      | 1          |
| 2   | Luis    | false    | 15.03.2019 | .net      | 3          |
| 3   | Bob     | false    | 22.11.2023 | Angular   | 5          |
| 4   | Patrick | true     | 15.12.2020 | React     | 2          |
| 5   | Roman   | true     | 13.10.2021 | Java      | 7          |

**SELECT** 
- *SELECT* * FROM Users
- *SELECT* u.title  FROM Users u
- *SELECT* *** FROM Users u Where u.isMaries = true  
**DML** 
*INSERT* INTO User (id, Name, isMaried , HiringDate, TechStack)
VALUES (1,'name','true','20.04.2018','.net');

*UPDATE* Users 
SET Title = 'Updated Title' WHERE Id = 2;

*DELETE*  FROM Users  where Id = 2

Якщо ми хочемо оновити або видалити данні за якоюсь умовою в таблиці і не використовуємо групування або сортування і в таблиці є кілька запитів які відповідають умові  то:
 - буде видалено перший запис який буде знайдено в таблиці зазвичай найстаріший запис тому-що по дефолту записи в базі зберігаються в порядку додавання 
- Якщо в  таблиці налаштовані індекси то буде видалений запис який буде знайдено перший за індексом 
- Якщо налаштоване блокування даних і область даних першого знайденого  не доступна для операції то операція буде здійснена для наступного запису знайденого в таблиці
- Якщо в бд використовується кеш то якщо запис в кеші буде відповідати умові то буде видалено цей запис

**AREGATION** - це процес обчислення одного значення на основі кількох елементів або записів даних

- *Count* - Обраховує кількість записів чи полів групи записів  
	Select Count() FROM Users u WHERE u.ISMaried = 'true'

- *SUM* - Обраховує суму записів та чи полів групи записів **Використовується тільки для числових значень**
	Select Sum(u.YearsOfExp) FROM Users u Where u.TechStack ='.net'

- *MIN* - Обраховує мінімальне значення записів чи полів групи записів **Використовується тільки для числових значень**
	SELECT MIN(u.YearsOfExp) FORM Users u Where u.TechStack ='.net'

- *AWG* - Обраховує середнє значення записів чи полів групи записів **Використовується тільки для числових значень**
	SELECT AWG(u.YearsOfExp) FORM Users u Where u.TechStack ='.net'

**Group by** Процес групування записів за одним полем часто використовується з агрегаці
тобто групування це створення групи записів за якимось полем 
	SELECT d.Title,COUNT(*) as titleCount
	FROM Users u
	GROUP By u.TechStack ='.net'
	
**Order by** Процес Сортування результатів з якимось полем
	 SELECT d.Title, MAX(d.CreationDate) AS LatestCreationDate
	FROM DiaryEntities d
	GROUP BY d.Title
	ORDER BY LatestCreationDate DESC;

**HAVING WHERE diverence**  Обидва оператори використовуються для фільтрації даних але Where фільтрує данні до того як групуються а Having після,
тобто Having фільтрує в кінці і лише по полям які були згруповані або агрегатовано тобто до них було використано наприклад Count() виводяться-повертаються  
 SELECT d.Title,d.Content ,MAX(d.CreationDate) as dateCreation
 FROM DiaryEntities d 
 Group BY d.Title ,d.Content
 having d.Content ='Go walk'

**TRANSACTION** *Все або нічого* 
	Транзакція - це набір упорядкованих операцій який зміню стан бд з одного узгодженого стану в інший упакованих в одну атомарну пачку 
	 Узгоджений стан це той який не порушує бізнес логіку системи


**Indexes** до 900 б

b-tree індекс 