### Create table
``` SQL
CREATE TABLE USERS(
	id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
	first_name VARCHAR(30),
	last_name VARCHAR(50),	
)
```
*В Postgres раніше для автоінкремента в основному використовувались ключове слово SERIAL, наразі для ID рекомендовано юзати GENERATED ALWAYS AS IDENTITY*

Приклад транзакції із побудови таблиць в Postgres:
``` SQL
BEGIN;  
  
CREATE TYPE order_status AS ENUM ('Created', 'Preparing', 'Shipping', 'Delivered', 'Done');  
CREATE TYPE payment_status AS ENUM ('Successful', 'Failed', 'Waiting');  
CREATE TYPE payment_method AS ENUM ('CreditCard', 'PayPal', 'Cash');  
  
CREATE TABLE Categories(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    Name VARCHAR(100) NOT NULL,  
    ParentId INT REFERENCES Categories(Id) NULL  
);  
  
CREATE TABLE Products(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    Name VARCHAR(255) NOT NULL,  
    Price DECIMAL(10, 2) NOT NULL,  
    StockQuantity INT DEFAULT 0,  
    ImageUrl varchar(255) NULL,  
    CreatedAt DATE DEFAULT(now()),  
    Description TEXT NULL,  
    CategoryId INT REFERENCES Categories(Id)  
);  
  
CREATE TABLE Customers(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    FirstName VARCHAR(100),  
    LastName varchar(100),  
    Email VARCHAR(255),  
    Phone VARCHAR(20),  
    Address VARCHAR(255),  
    CreatedAt timestamptz DEFAULT (now())  
);  
  
CREATE TABLE Orders(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY ,  
    CustomerId INT REFERENCES Customers(Id),  
    OrderDate timestamptz DEFAULT now(),  
    TotalAmount DECIMAL(10, 2) CHECK ( TotalAmount >= 0 ),  
    Status order_status DEFAULT ('Created')  
);  
  
CREATE TABLE OrderItems(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    OrderId INT REFERENCES Orders(Id),  
    ProductId INT REFERENCES Products(Id),  
    Quantity INT NOT NULL  CHECK ( Quantity > 0 ),  
    UnitPrice DECIMAL(10,2)  
);  
  
CREATE TABLE Payments(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    OrderId INT REFERENCES Orders(Id),  
    PaymentDate timestamptz,  
    Amount DECIMAL(10,2) CHECK ( Amount >= 0 ),  
    PaymentMethod payment_method NOT NULL,  
    Status payment_status DEFAULT ('Waiting')  
);  
  
CREATE TABLE ShoppingCart(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    CustomerId INT REFERENCES Customers(Id),  
     CreatedAt timestamptz DEFAULT (now())  
);  
  
CREATE TABLE CartItems(  
    Id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  
    CartId INT REFERENCES ShoppingCart(Id),  
    ProductId INT REFERENCES Products(Id),  
    Quantity INT CHECK ( Quantity > 0 )  
);  
  
END;
```
### Create type
Щоб зробити свою enum треба створити власний тип ([[PostgreSQL Data Types]]) від enum, це робиться таким чином:
``` SQL
CREATE TYPE order_status AS ENUM ('Created', 'Preparing', 'Shipping', 'Delivered', 'Done')
```

### Insert
Для заповнення таблиць даними в SQL використовується команда `INSERT`. Ось приклад запиту для заповнення даними попередньо створеної таблиці:
``` SQL
BEGIN;  
  
INSERT INTO Categories (Name, ParentId) VALUES  
('T-Shirts', NULL),  
('Mugs', NULL),  
('Hats', NULL),  
('Stickers', NULL),  
('Hoodies', NULL);  
  
INSERT INTO Products (Name, Price, StockQuantity, ImageUrl, Description, CategoryId) VALUES  
('Company Logo T-Shirt', 19.99, 100, 'https://example.com/t-shirt.jpg', 'Comfortable cotton t-shirt with the company logo.', 1),  
('Classic Coffee Mug', 9.99, 50, 'https://example.com/mug.jpg', 'White mug with a printed logo.', 2),  
('Baseball Cap', 15.99, 30, 'https://example.com/hat.jpg', 'Stylish cap with embroidered logo.', 3),  
('Vinyl Sticker', 2.99, 200, 'https://example.com/sticker.jpg', 'Durable vinyl sticker with company branding.', 4),  
('Company Logo Hoodie', 39.99, 40, 'https://example.com/hoodie.jpg', 'Warm hoodie with a stylish company logo.', 5);  
  
INSERT INTO Customers (FirstName, LastName, Email, Phone, Address) VALUES  
('John', 'Doe', 'john.doe@example.com', '1234567890', '123 Main St, Hometown, USA'),  
('Jane', 'Smith', 'jane.smith@example.com', '0987654321', '456 Elm St, Hometown, USA'),  
('Sam', 'Brown', 'sam.brown@example.com', '1122334455', '789 Pine St, Hometown, USA');  
  
INSERT INTO Orders (CustomerId, TotalAmount, Status) VALUES  
(1, 29.98, 'Created'),  
(2, 19.99, 'Preparing'),  
(3, 55.98, 'Shipping');  
   
INSERT INTO OrderItems (OrderId, ProductId, Quantity, UnitPrice) VALUES  
(1, 1, 1, 19.99),  
(1, 2, 1, 9.99),  
(2, 1, 1, 19.99),  
(3, 1, 2, 19.99),  
(3, 5, 1, 39.99);  
  
INSERT INTO Payments (OrderId, PaymentDate, Amount, PaymentMethod, Status) VALUES  
(1, now(), 29.98, 'CreditCard', 'Successful'),  
(2, now(), 19.99, 'PayPal', 'Successful'),  
(3, now(), 59.98, 'CreditCard', 'Waiting');  
  
INSERT INTO ShoppingCart (CustomerId) VALUES  
(1),  
(2),  
(3);  
  
INSERT INTO CartItems (CartId, ProductId, Quantity) VALUES  
(1, 1, 1),  
(1, 2, 2),  
(2, 3, 1),  
(3, 4, 5);  
  
END;
```
### Select 
Для вибірки інформації з таблиць використовується команда `SELECT`. Разом з нею можуть використовуватись багато різних команд для доповнення отримуваного значення. Приклад запиту для отримання максимально повної інформації про замовлення:
``` SQL
SELECT
    o.Id AS OrderId,
    o.OrderDate,
    o.TotalAmount,
    o.Status AS OrderStatus,
    c.FirstName || ' ' || c.LastName AS CustomerName,
    pr.Name AS ProductName,
    oi.Quantity,
    oi.UnitPrice
FROM Orders o
JOIN Customers c ON o.CustomerId = c.Id
JOIN OrderItems oi ON o.Id = oi.OrderId
JOIN Products pr ON oi.ProductId = pr.Id
ORDER BY o.OrderDate DESC;

```
В даному запиті окрім відомого `SELECT` використовуються також ключові слова `JOIN` для доєднання інформації із сусідніх таблиць (по дефолту `JOIN` означає виклик `INNER JOIN`), та ORDER BY для сортування даних за певним полем, у нашому випадку за датою і `DEST`, тобто за спаданням значення, тобто спочатку найновіші.
`||` - оператор означає конкатенацію рядків.
### Keys 
#### Primary key
Primary key - це ключ, який є єдиним ідентифікатором таблиці, він має бути унікальним для кожного запису. `id INT PRIMARY KEY`
### Foreign key
Foreign key - ключ, який використовуєтсья для зв'язку між таблицями. 
Ось приклад зовнішнього ключа **1 to N**:  `user_id INT REFERENCES users(id)`
Щоб зробити **1 to 1** треба написати UNIQUE додатково: `user_id INT UNIQUE REFERENCES users(id)` .
Для оглошення зв'язку N to N потрібно створювати додаткову сутність:
``` SQL
CREATE TABLE StudentCourses (
    StudentId INT,
    CourseId INT,
    PRIMARY KEY (StudentId, CourseId), 
    FOREIGN KEY (StudentId) REFERENCES Students(Id) ON DELETE CASCADE,
    FOREIGN KEY (CourseId) REFERENCES Courses(Id) ON DELETE CASCADE
);
```
Можна також помітити, що в зв'язуючої таблиці primary key складається із 2 колонок, які є референсами на 2 інші таблиці.
### Update table
UPDATE використовується для оновлення записів в таблиці, переважно це роблять за умовами, тобто наприклад `UPDATE Products SET Price = 2 WHERE Id = 2`.
Також UPDATE можна використовувати разом з `FROM`, а також підзапитами.
``` SQL
UPDATE Orders
SET TotalAmount = (
    SELECT SUM(UnitPrice * Quantity)
    FROM OrderItems
    WHERE OrderItems.OrderId = Orders.Id
)
WHERE Status = 'Created';
```
Цікавою особливістю є те, що `UPDATE` можна використовувати із ключовим словом `RETURNING`, завдяки чому повертати деякі поля оновленої таблиці.
``` SQL
UPDATE Products
SET Price = Price * 1.10
WHERE CategoryId = 1
RETURNING Id, Name, Price;
```
Окрім того, `RETURNING` можна писати разом з `DELETE`, `INSERT`, `MERGE`
### Alter table
`ALTER TABLE` це запит, який використовується для зміни структури таблиці. 
Наприклад, так, за допомогою ALTER TABLE можна:
- додати нову колонку до бд `ALTER TABLE products ADD COLUMN Discount DECIMAL (5,2) DEFAULT 0.0`
- видалити колонку `ALTER TABLE products DROP COLUMN Discount`
- зміна типу колонки `ALTER TABLE products ALTER COLUMN Discount TYPE INT`
- зміна властивості колонки `ALTER TABLE products ALTER COLUMN Discount SET(або DROP) NOT NULL`
- перейменування колонки `ALTER TABLE products RENAME COLUMN Discount TO DiscountRate`
- перейменування таблиці `ALTER TABLE products RENAME TO ProductCatalog`
- зміна послідовності PK: `ALTER SEQUENCE Products_Id_seq RESTART WITH 1000;`
- додавання або видалення FK: `ALTER TABLE table_name ADD CONSTRAINT fk_name FOREIGN KEY (column_name) REFERENCES other_table(column_name);`
	`ALTER TABLE Orders DROP CONSTRAINT fk_customer;`
### Delete table
To delete table from the database is used query `DROP TABLE `
To remove records from the tables are used `DELETE FROM` and `TRUNCATE TABLE` queries. The only difference between them is in that `DELETE FROM` allows to delete records by condition, but `TRUCATE` removes all the rows immediately.

У контексті запитів в Postgres варто також звернути увагу на [[Stored Procedures]] та [[Індекси]] та [[Транзакції]] які застосовувались при написанні деяких запитів в цій нотатці
