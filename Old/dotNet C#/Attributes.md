Атрибути - це класи, які можна встановити як додаткові умови для інших різних об'єктів. 
Щоб оголосити атрибут, потрібно створити клас і успадкувати його від класу Attribute. Усі дії, які будуть виконуватись атрибутом із об'єктом описуються у конструкторі. Щоб вказати, до якого типу об'єкту(клас/метод) потрібно застосовуати атрибут, до класу атрибута потрібно вказати атрибут `[AttributeUsage]` 
Синтаксис оголошення і використання:
``` C#
using System;

// Створення власного атрибута
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true)] // Вказуємо, де можна використовувати атрибут
public class MyCustomAttribute : Attribute
{
    public string Description { get; set; } // Іменований параметр
    public int Version { get; } // Позиційний параметр

    public MyCustomAttribute(int version) // Конструктор з позиційним параметром
    {
        Version = version;
    }
}

// Використання атрибута
[MyCustomAttribute(1, Description = "Опис класу")]
[MyCustomAttribute(2, Description = "Другий опис")]
public class MyClass
{
    [MyCustomAttribute(3, Description = "Опис методу")]
    public void MyMethod()
    {
        // ...
    }
}

public class Example
{
    public static void Main(string[] args)
    {
        Type type = typeof(MyClass);
        object[] attributes = type.GetCustomAttributes(typeof(MyCustomAttribute), true);

        foreach (MyCustomAttribute attribute in attributes)
        {
            Console.WriteLine($"Version: {attribute.Version}, Description: {attribute.Description}");
        }
    }
}
```
