---
tags: [architecture, clean-architecture, solid, dotnet]
aliases: [Clean Architecture]
---

# Clean Architecture

Clean Architecture — це архітектурний підхід, запропонований Робертом Мартіном (Uncle Bob), який фокусується на розділенні відповідальностей та незалежності бізнес-логіки від зовнішніх деталей (фреймворків, БД, UI).

## Основні принципи
- **Незалежність від фреймворків** — бізнес-логіка не залежить від конкретного фреймворку
- **Тестованість** — бізнес-правила можна тестувати без UI, БД чи веб-сервера
- **Незалежність від UI** — UI можна змінити без зміни бізнес-логіки
- **Незалежність від БД** — можна замінити Oracle на SQL Server, MongoDB тощо

## Шари (від внутрішнього до зовнішнього)
1. **Domain (Entities)** — бізнес-об'єкти та правила
2. **Application (Use Cases)** — логіка додатку, інтерфейси репозиторіїв
3. **Infrastructure** — реалізації репозиторіїв, доступ до БД ([[EF Core additional]]), зовнішні сервіси
4. **Presentation** — контролери, [[Middlewares]], [[Filters]]

Залежності спрямовані всередину: зовнішні шари залежать від внутрішніх, але не навпаки. Це досягається через [[Di .net applications|Dependency Injection]] та інтерфейси (патерн [[Design patterns#Міст (*Bridge*)|Bridge]]).

## Див. також
- [[Monolithic Architecture]]
- [[Microservice Architecture]]
- [[Design patterns]]
- [[Di .net applications]]
