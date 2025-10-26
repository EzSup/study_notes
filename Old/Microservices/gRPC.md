gRPC (Google Remote Procedure Call) - фреймворк, побудований поверх HTTP/2.0, який використовує протобаф для опису сервісів.
Патерни взаємодії в gRPC:
1. Server to client streaming (клієнт робить один запит і отримує багато пакетів у відповідь від сервера)
2. Client to server streaming
3. Bi-directional streaming
4. Unary RPC (фактично як звичайний RESTful API)
![[Pasted image 20241228151300.png]]
Формати серіалізації в gRPC:
1. [[Protobuff]]
2. JSON
