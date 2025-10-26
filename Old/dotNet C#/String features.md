
int number = 42;
string formatted = number.ToString("D5"); // "00042"


### 1. **Форматування чисел**

- **Додати провідні нулі:**
    
    csharp
    
    Копіювати код
    
    `int number = 42; string formatted = number.ToString("D5"); // "00042"`
    
- **Форматувати як грошове значення:**
    
    csharp
    
    Копіювати код
    
    `double price = 1234.56; string currency = price.ToString("C"); // "$1,234.56" (або інша валюта залежно від культури)`
    
- **Формат із роздільниками тисяч:**
    
    csharp
    
    Копіювати код
    
    `int bigNumber = 1234567; string formattedNumber = bigNumber.ToString("N0"); // "1,234,567"`
    

---

### 2. **Форматування дат і часу**

- Вивести поточний час:
    
    csharp
    
    Копіювати код
    
    `DateTime now = DateTime.Now; string time = now.ToString("HH:mm:ss"); // "14:35:10" (24-годинний формат) string date = now.ToString("yyyy-MM-dd"); // "2024-11-17"`
    
- Формат із текстом:
    
    csharp
    
    Копіювати код
    
    `string dateWithDay = now.ToString("dddd, MMMM dd, yyyy"); // "Sunday, November 17, 2024"`
    

---

### 3. **Комбіновані формати**

Іноді можна створювати комбіновані рядки:

csharp

Копіювати код

`int hours = 7; int minutes = 5; int seconds = 12;  string fullTime = $"{hours:D2}:{minutes:D2}:{seconds:D2}"; // "07:05:12" Console.WriteLine(fullTime);`

---

### 4. **Динамічна культура (локалізація)**

Якщо потрібно форматувати значення відповідно до конкретної мови чи регіону:

csharp

Копіювати код

`double price = 1234.56; var culture = new System.Globalization.CultureInfo("uk-UA"); // Українська культура string formattedPrice = price.ToString("C", culture); // "1 234,56 ₴" Console.WriteLine(formattedPrice);`

---

### 5. **Час, як у твоєму прикладі**

Ось повноцінний приклад із годинами та хвилинами:

csharp

Копіювати код

`int hours = 7; int minutes = 5;  string time = $"{hours:D2}:{minutes:D2}"; Console.WriteLine($"Зараз час: {time}"); // "Зараз час: 07:05"`