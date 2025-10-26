>Stream is the abstract base class of all streams. A stream is an abstraction of a sequence of bytes, such as a file, an input/output device, an inter-process communication pipe, or a TCP/IP socket. The Stream class and its derived classes provide a generic view of these different types of input and output, and isolate the programmer from the specific details of the operating system and the underlying devices.

**Stream**(стрім або потік) це просто рівень абстракції, який дає можливість чиатти/записувати потоки байтів з/e файл, різні девайси, черхи та TCP/IP сокети 

![[Pasted image 20250209203706.png]]
## Stream methods
1. CanRead() - чи читабельний стрім;
2. CanWrite() - чи писабельний стрім;
3. Read/ReadByte/ReadAsync;
4. Write/WriteByte/WriteAsync;
5. CanSeek - Seek означає що ти можеш встановити позицію всередині стріма. Ти можеш шукати від `current position`, `beginning` або `the end`;
6. Dispose - стріми реалізують інтерфейс IDisposable
## Stream VS byte array
Ось приклади коду копіювання файла:
1. Через byte[]:
``` C#
using System.Diagnostics;  
  
const string InputFile = "../input/video.mp4";  
const string OutputFile = "../output/video.mp4";  
const int RepeatTime = 50;  
  
var stopwatch = new Stopwatch();  
var result = new List<TimeSpan>();  
for (int i = 0; i < RepeatTime; i++)  
{  
File.Delete(OutputFile);  
stopwatch.Reset();  
  
stopwatch.Start();  
var content = File.ReadAllBytes(InputFile);  
File.WriteAllBytes(OutputFile, content);  
stopwatch.Stop();  
  
result.Add(stopwatch.Elapsed);  
}  
  
Console.WriteLine($"Average: {result.Average(x => x.TotalMilliseconds)}ms");
```
2. Через Stream
``` C#
using System.Diagnostics;  
  
const string InputFile = "../input/video.mp4";  
const string OutputFile = "../output/video.mp4";  
const int RepeatTime = 50;  
  
var stopwatch = new Stopwatch();  
var result = new List<TimeSpan>();  
for (int i = 0; i < RepeatTime; i++)  
{  
File.Delete(OutputFile);  
stopwatch.Reset();  
  
stopwatch.Start();  
using var input = File.OpenRead(InputFile);  
using var output = File.OpenWrite(OutputFile);  
input.CopyTo(output);  
stopwatch.Stop();  
  
result.Add(stopwatch.Elapsed);  
}  
  
Console.WriteLine($"Average: {result.Average(x => x.TotalMilliseconds)}ms");
```

Результати виконання цих 2-х варіантів:
```
Byte Array - 557.25 ms  
Stream - 91.22 ms
```
Використовувана пам'ять варіанту з byte[]: 
![[Pasted image 20250209205524.png]]
Використана пам'ять варіанту зі Stream:
![[Pasted image 20250209205651.png]]
Стріми зберігають в буфер тільки ту частину даних, яку в момент пересилають, і одразу його віддають, записують наступну частину і так далі. Тому ми і спостерігаємо таку значну оптимізацію пам'яті при переході на стріми.
## So _prefer_ streams when:
- you do not need to work with entire stream content;
- you are working with large streams of data.
## Stream Reader and Writer
Потокові(стрімові) методи дозволяють читати/записувати, використовуючи лише байтовий масив як буфер. Для перетворення вмісту в/з деякого текстового кодування ви можете використовувати вбудовані методи Encoding (GetBytes для перетворення рядка в байтовий масив і GetString для зворотної операції). Але є й кращі рішення - класи StreamReader та StreamWriter.
Код програми, яка прочитає файл, запише в консоль його вміст і скопіює його в інший файл, щоб побачити, як використовувати наведені вище класи:

```C#
using System.Text;  
  
const string InputFile = "../input/file.txt";  
const string OutputFile = "../output/file.txt";  
  
using var input = File.OpenRead(InputFile);  
using var reader = new StreamReader(input, Encoding.UTF8);  
using var output = File.OpenWrite(OutputFile);  
using var writer = new StreamWriter(output, Encoding.UTF8);  
  
while (!reader.EndOfStream)  
{  
	var result = reader.ReadLine();  
	Console.WriteLine(result);  
	writer.WriteLine(result);  
}
```

### Methods
**StreamReader** provides the following methods:

- **Read/ReadAsync** to read the next character from a stream;
- **ReadBlock**/**ReadBlockAsync** to read file content to buffer from a stream;
- **ReadLine**/**ReadLineAsync** to read the whole line from a stream;
- **ReadToEnd**/**ReadToEndAsync** to read all stream’s content.

StreamWriter provides the following methods:

- **Write/WriteAsync** to write a single character to stream;
- **WriteLine**/**WriteLineAsync** to write a line to stream.

Both **StreamReader** and **StreamWriter** are **disposable** resources.