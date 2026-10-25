# NestJS `AppModule`-এ gRPC Client Configuration ব্যাখ্যা

এই `AppModule`-এর কোডটি NestJS-এ একটি gRPC client configure করার জন্য ব্যবহৃত হচ্ছে। এখানে `ClientsModule`, `Transport.GRPC`, `protoPath` এবং `url`-এর মাধ্যমে NestJS-কে জানানো হচ্ছে কীভাবে `InventoryService`-এর সঙ্গে gRPC ব্যবহার করে যোগাযোগ করতে হবে।

## ১. `ClientsModule` এবং `Transport` import

```ts
import { ClientsModule, Transport } from '@nestjs/microservices';
```

এখানে দুটি জিনিস import করা হয়েছে।

**`ClientsModule`**

- এটি NestJS-এর Microservices package-এর অংশ।
- অন্য microservice-এর সঙ্গে যোগাযোগ করার জন্য client তৈরি ও configure করতে ব্যবহৃত হয়।
- এর মাধ্যমে তৈরি client-কে NestJS dependency injection-এর সাহায্যে অন্য class-এ ব্যবহার করা যায়।

**`Transport`**

- এটি communication-এর transport type নির্ধারণ করে।
- NestJS-এ TCP, Redis, MQTT, gRPC-সহ বিভিন্ন transport রয়েছে।
- এই কোডে `Transport.GRPC` ব্যবহার করা হয়েছে।

সহজভাবে: `ClientsModule` client configure করে, আর `Transport.GRPC` বলে দেয় যে যোগাযোগের মাধ্যম হবে gRPC।

## ২. `join` import

```ts
import { join } from 'path';
```

- `path` হলো Node.js-এর built-in module।
- `join()` function একাধিক path অংশকে যুক্ত করে একটি path তৈরি করে।
- Windows ও Linux-এর মতো ভিন্ন operating system-এ path separator ঠিকভাবে সামলাতে এটি সাহায্য করে।

উদাহরণ:

```ts
join(process.cwd(), 'libs/proto/inventory.proto')
```

যদি project root হয়:

```
C:\Users\User\Desktop\wasuTech-gRPC
```

তাহলে তৈরি হওয়া path হবে:

```
C:\Users\User\Desktop\wasuTech-gRPC\libs\proto\inventory.proto
```

`process.cwd()` বর্তমান working directory ফেরত দেয়। সাধারণত প্রজেক্টের root directory থেকে কমান্ড চালালে এটি সেই directory হবে।

## ৩. `ClientsModule.register([...])`

```ts
ClientsModule.register([
  {
    name: 'INVENTORY_PACKAGE',
    transport: Transport.GRPC,
    options: {
      package: 'inventory',
      protoPath: join(process.cwd(), 'libs/proto/inventory.proto'),
      url: 'localhost:50051',
    },
  },
])
```

এটি পুরো configuration-এর মূল অংশ। এবার ভেঙে দেখা যাক।

### `ClientsModule.register()`

- `register()`-এর মাধ্যমে client configuration নিবন্ধন করা হয়।
- এখানে একটি configuration object array-এর মধ্যে দেওয়া হয়েছে।
- এই configuration অনুসারে NestJS একটি gRPC client তৈরি করার জন্য প্রয়োজনীয় তথ্য পায়।

### `name: 'INVENTORY_PACKAGE'`

```ts
name: 'INVENTORY_PACKAGE',
```

এটি client-এর injection token বা পরিচয়।

পরবর্তীতে controller বা service-এ এই token ব্যবহার করে client নেওয়া যায়।

উদাহরণ:

```ts
import { Controller, Inject } from '@nestjs/common';
import { ClientGrpc } from '@nestjs/microservices';

@Controller()
export class AppController {
  constructor(
    @Inject('INVENTORY_PACKAGE')
    private readonly inventoryClient: ClientGrpc,
  ) {}
}
```

এখানে `@Inject('INVENTORY_PACKAGE')` NestJS-কে বলে, নিবন্ধিত `INVENTORY_PACKAGE` client-টি এখানে inject করতে হবে।

### `transport: Transport.GRPC`

```ts
transport: Transport.GRPC,
```

এটি বলে দেয় যে এই client gRPC transport ব্যবহার করবে।

অর্থাৎ, NestJS-কে TCP transport বা অন্য কোনো transport নয়, gRPC-এর নিয়ম অনুসারে যোগাযোগ করতে হবে।

### `options.package: 'inventory'`

```ts
options: {
  package: 'inventory',
}
```

এটি `.proto` ফাইলে ঘোষিত package-এর নামের সঙ্গে মিলতে হবে।

`inventory.proto` ফাইলে আছে:

```proto
package inventory;
```

তাই এখানে `'inventory'` দেওয়া হয়েছে।

**কেন দরকার?** gRPC client-কে `.proto` ফাইলের সঠিক package-এর service খুঁজে পেতে এই তথ্য ব্যবহার করতে হয়।

### `options.protoPath`

```ts
protoPath: join(process.cwd(), 'libs/proto/inventory.proto'),
```

এটি `.proto` ফাইলের path নির্ধারণ করে।

Project structure এমন হলে:

```
wasuTech-gRPC/
├── libs/
│   └── proto/
│       └── inventory.proto
├── src/
│   ├── app.module.ts
│   └── app.controller.ts
└── package.json
```

তাহলে `protoPath`-এ দেওয়া path-টি সঠিক ফাইলের দিকে নির্দেশ করবে।

এই `.proto` ফাইলেই `InventoryService`, `CheckStock`, `StockRequest` এবং `StockResponse`-এর মতো declaration রয়েছে।

### `options.url: 'localhost:50051'`

```ts
url: 'localhost:50051',
```

এটি gRPC server-এর address ও port নির্ধারণ করে।

- `localhost` — একই কম্পিউটারে চলা server-কে বোঝায়।
- `50051` — server-এর port number।

Inventory gRPC server-কে অবশ্যই এই address ও port-এ listen করতে হবে। অন্য port-এ চললে এখানে সেই port দিতে হবে।

Docker বা আলাদা কম্পিউটারে server চললে `localhost`-এর পরিবর্তে উপযুক্ত hostname বা service name প্রয়োজন হতে পারে।

## ৪. পুরো বিষয়টি একসঙ্গে বুঝে নাও

```
NestJS Client Module
        │
        ▼
ClientsModule.register()
        │
        ▼
gRPC Configuration
(INVENTORY_PACKAGE · Transport.GRPC · inventory.proto)
        │
        ▼
Inventory gRPC Server
(localhost:50051)
```

Client এই configuration ব্যবহার করে server-এর RPC method-এ request পাঠাতে পারে।

> মনে রাখবে, শুধু `ClientsModule.register()` লিখলেই `CheckStock` method স্বয়ংক্রিয়ভাবে কল হবে না। এরপর client থেকে service নেওয়া এবং RPC method কল করার কোডও লিখতে হবে।
