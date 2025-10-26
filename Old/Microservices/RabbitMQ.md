**RabbitMQ** - це брокер повідомлень. Його основна мета - приймати і віддавати повідомлення. Його можна уявляти собі, як поштове відділення: коли Ви кидаєте лист у скриньку, Ви можете бути впевнені, що рано чи пізно листоноша доставить його адресату. У цій аналогії RabbitMQ є одночасно і поштовою скринькою, і поштовим відділенням, і листоношею.
Найбільша відмінність RabbitMQ від поштового відділення в тому, що він не має справи з паперовими конвертами - RabbitMQ приймає, зберігає і віддає бінарні дані - повідомлення.
# Термінологія
## Producer
**Producer (постачальник)** - програма, що надсилає повідомлення.
## Consumer
**Consumer (підписник)** - програма, що приймає повідомлення. Зазвичай передплатник перебуває в стані очікування повідомлень. 
## Queue
**Queue (черга)** - ім'я «поштової скриньки». Вона існує всередині RabbitMQ. Хоча повідомлення проходять через RabbitMQ і додатки, зберігаються вони тільки в чергах. Черга не має обмежень на кількість повідомлень, вона може прийняти як завгодно велику їхню кількість - можна вважати її нескінченним буфером. Будь-яка кількість постачальників може надсилати повідомлення в одну чергу, також будь-яка кількість підписників може отримувати повідомлення з однієї черги
## Exchange
**Exchange(обмінник)** - отримує повідомлення від продюсерів і направляє їх у черги згідно правил. (повідомлення можна направляти напряму в чергу, без exchange)
- Direct Exchange - доставляє повідомлення в конкретну чергу на основі routing key. Дефолтний exchange є direct.
- Topic Exchange - направляє повідомлення на основі **патерну** routing key.
- Fanout Exchange - розсилає повідомлення у всі прив'язані до цього exchange черги. Ігнорує routing key. Підходить для broadcast сценаріїв.
- Headers Exchange - використовує атрибути заголовків для маршрутизації. Ігнорує routing key. Підходить для складної маршрутизації.
## Binding
Binding - це зв'язок між exchange і queue. Вони визначають, яким чином Exchange має розподіляти повідомлення у Queue.
## Routing key
**Routing key** - атрибут повідомлення, який визначає, як exchange буде маршрутизувати повідомлення до черг. Це фактично адреса чи мітка, яка вказує, куди повідомлення має бути доставлене.

```C#
// Публікація повідомлення
channel.BasicPublish(
    exchange: "order_exchange",
    routingKey: "orders.europe.created",
    body: messageBody
);

// Прив'язка черги до exchange
channel.QueueBind(
    queue: "european_orders_queue",
    exchange: "order_exchange",
    routingKey: "orders.europe.*"
);
```

Правильне використання routing key дозволяє будувати гнучкі системи маршрутизації повідомлень, де різні компоненти можуть підписуватися лише на ті повідомлення, які їм потрібні для роботи.
# Code samples CSharp
## Producer

```C#
using RabbitMQ.Client;  
using System.Text;  
  
namespace RabbitSender;  
  
class Program  
{  
    static async Task Main(string[] args)  
    {        ConnectionFactory factory = new ConnectionFactory();  
        factory.Uri = new Uri("amqp://guest:guest@localhost:5672");  
        factory.ClientProvidedName = "Rabbit Sender App";  
        IConnection cnn =  await factory.CreateConnectionAsync();  
        IChannel channel =  await cnn.CreateChannelAsync();  
  
        string exchangeName = "DemoExchange";  
        string routingKey = "demo-routing-key";  
        string queueName = "DemoQueue";  
        await channel.ExchangeDeclareAsync(exchangeName, ExchangeType.Direct);  
        await channel.QueueDeclareAsync(queueName, durable:true, exclusive:false, autoDelete: false);  
        await channel.QueueBindAsync(queueName, exchangeName, routingKey);  
  
        for (int j = 3; j > 0; j--)  
        {            for (int i = new Random().Next(97,152); i > 0; i--)  
            {                byte[] messageBodyBytes = Encoding.UTF8.GetBytes($"Hello Rabbit! #{i}");  
                        await channel.BasicPublishAsync(exchange: exchangeName, routingKey: routingKey, body: messageBodyBytes);  
            }            Thread.Sleep(10000);  
        }  
        await channel.CloseAsync();  
        await cnn.CloseAsync();  
    }}
```

## Reciever

```C#
using RabbitMQ.Client;  
using RabbitMQ.Client.Events;  
using System.Text;  
  
namespace RabbitReciever1;  
  
class Program  
{  
    static async Task Main(string[] args)  
    {        ConnectionFactory factory = new ConnectionFactory();  
        factory.Uri = new Uri("amqp://guest:guest@localhost:5672");  
        factory.ClientProvidedName = "Rabbit Reciever1 App";  
        IConnection cnn =  await factory.CreateConnectionAsync();  
        IChannel channel =  await cnn.CreateChannelAsync();  
  
        string exchangeName = "DemoExchange";  
        string routingKey = "demo-routing-key";  
        string queueName = "DemoQueue";  
        await channel.ExchangeDeclareAsync(exchangeName, ExchangeType.Direct);  
        await channel.QueueDeclareAsync(queueName, durable:false, exclusive:false, autoDelete: false);  
        await channel.QueueBindAsync(queueName, exchangeName, routingKey);  
        await channel.BasicQosAsync(0, 1, false);  
  
        var consumer = new AsyncEventingBasicConsumer(channel);  
        consumer.ReceivedAsync += async (sender, args) =>  
        {  
            Task.Delay(TimeSpan.FromMilliseconds(200)).Wait();  
            var body = args.Body.ToArray();  
            string message = Encoding.UTF8.GetString(body);  
            Console.WriteLine($"Received message is: {message}");  
            await channel.BasicAckAsync(args.DeliveryTag, false);  
        };  
        string consumerTag = await channel.BasicConsumeAsync(queueName, false, consumer);  
  
        Console.ReadLine();  
  
        await channel.BasicCancelAsync(consumerTag);  
        await channel.CloseAsync();  
        await cnn.CloseAsync();  
    }}
```