Protobuff(Protocol buffers) - формат серіалізації даних, запропонований Google на заміну XML. Використовується для визначення схем в мікросервісах, при роботі із протоколом gRPC.
В gRPC в `.proto` файлах описують "інтерфейс" певного сервіса, тобто набір дій і очікуваних структур відповідей від такого сервіса. 
Прото-файли є мовно-нейтральними, тобто сумісними із усіма мовами програмування, і автоматично парсяться у код відповідної мови, яка звертається до порібних сервісів.
#### Приклад `.proto`-файлу:
``` proto
syntax = "proto3";

service UserService {
  rpc GetUserById (GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  int32 user_id = 1;
}

message GetUserResponse {
  int32 user_id = 1;
  string name = 2;
  string email = 3;
}

```
