# NestJS Order Service Controller (gRPC Client) ব্যাখ্যা

এই কোডটি হলো Order Service-এর REST API Controller, যেটি gRPC ব্যবহার করে `InventoryService` থেকে পণ্যের stock-এর তথ্য সংগ্রহ করে। এরপর stock আছে কি না, তার ভিত্তিতে client-কে HTTP response দেয়।

## ১. `Observable` এবং `lastValueFrom` import

```ts
import { Observable, lastValueFrom } from 'rxjs';
```

এখানে RxJS থেকে দুটি জিনিস import করা হয়েছে।

### `Observable` কী?

`Observable` হলো এমন একটি object, যার মাধ্যমে সময়ের সঙ্গে এক বা একাধিক asynchronous value পাওয়া যেতে পারে।

gRPC client-এর `checkStock()` method একটি `Observable` ফেরত দেয়।

উদাহরণ:

```ts
this.inventoryService.checkStock({ productId: '123' });
```

এটি সরাসরি stock-এর ফলাফল না দিয়ে একটি `Observable` ফেরত দেয়। সেই Observable থেকে response পাওয়ার জন্য subscribe করতে হয় অথবা উপযুক্ত conversion ব্যবহার করতে হয়।

### `lastValueFrom` কী?

```ts
lastValueFrom(observable)
```

এটি একটি Observable-এর শেষ value পাওয়ার জন্য অপেক্ষা করে এবং সেটিকে `Promise`-এ রূপান্তর করে।

এখানে gRPC request-এর response পাওয়ার জন্য এটি ব্যবহার করা হয়েছে।

সহজভাবে:

- `Observable` → RxJS-এর asynchronous data stream।
- `lastValueFrom()` → Observable-এর শেষ value-কে Promise-এর মাধ্যমে পাওয়ার উপায়।

gRPC-এর সাধারণ request-response ব্যবহারে একটি response আসার পর Observable complete হয়, তাই এখানে `lastValueFrom()` উপযুক্ত।

## ২. `ClientGrpc` import

```ts
import { ClientGrpc } from '@nestjs/microservices';
```

`ClientGrpc` হলো NestJS-এর gRPC client interface।

এটি ব্যবহার করে `.proto` ফাইলে সংজ্ঞায়িত service-এর client-side method-গুলো access করা যায়।

যেমন:

```ts
this.client.getService<InventoryService>('InventoryService');
```

এখানে `getService()` দিয়ে `InventoryService`-এর client proxy নেওয়া হচ্ছে।

> মনে রাখবে: `ClientGrpc` নিজে stock পরীক্ষা করে না। এটি gRPC server-এর সঙ্গে যোগাযোগ করার client-side ব্যবস্থা দেয়।

## ৩. `interface InventoryService`

```ts
interface InventoryService {
  checkStock(data: { productId: string }): Observable<any>;
}
```

এটি TypeScript-এর interface। এটি জানিয়ে দেয়, `InventoryService` থেকে কী ধরনের method আশা করা হচ্ছে।

ভেঙে দেখা যাক:

- `interface InventoryService` — interface-এর নাম।
- `checkStock` — method-এর নাম।
- `data: { productId: string }` — method-এ একটি object পাঠাতে হবে, যেখানে `productId` হবে string।
- `Observable<any>` — method-টি একটি Observable ফেরত দেবে।

উদাহরণ:

```ts
this.inventoryService.checkStock({
  productId: '123',
});
```

এই interface-টি `.proto` ফাইলের সঙ্গে মিল রেখে লেখা হয়েছে:

```proto
service InventoryService {
    rpc CheckStock (StockRequest) returns (StockResponse);
}
```

তবে interface শুধু TypeScript-কে method-এর structure জানায়। এটি নিজে gRPC method তৈরি করে না।

আরও ভালো type safety-এর জন্য `any`-এর পরিবর্তে response interface ব্যবহার করা যায়:

```ts
interface StockResponse {
  availableQuantity: number;
  inStock: boolean;
}

interface InventoryService {
  checkStock(data: { productId: string }): Observable<StockResponse>;
}
```

## ৪. `implements OnModuleInit`

```ts
export class AppController implements OnModuleInit {
```

এখানে তিনটি অংশ আছে।

- `export class AppController` — `AppController` class-টিকে অন্য ফাইল থেকে ব্যবহার করার উপযোগী করে।
- `implements OnModuleInit` — class-টি NestJS-এর `OnModuleInit` lifecycle interface বাস্তবায়ন করছে।
- `OnModuleInit` — NestJS-এর lifecycle hook; module-এর dependencies initialize হওয়ার পর `onModuleInit()` চালানো হয়।

এখানে এর উদ্দেশ্য হলো, controller প্রস্তুত হওয়ার সময় gRPC service client-টি initialize করা।

## ৫. `private inventoryService!: InventoryService;`

```ts
private inventoryService!: InventoryService;
```

এই লাইনটি একটু গুরুত্বপূর্ণ।

- `private` — class-এর ভেতরে এই property ব্যবহার করার জন্য নির্ধারিত।
- `inventoryService` — property-এর নাম।
- `: InventoryService` — property-টি উপরে সংজ্ঞায়িত interface-এর structure অনুসরণ করবে।
- `!` — TypeScript-এর definite assignment assertion। এটি TypeScript-কে জানায় যে property-টি constructor-এ সরাসরি initialize না হলেও পরে initialize করা হবে।

এখানে property-টি `onModuleInit()`-এর মধ্যে initialize করা হবে।

```ts
onModuleInit() {
  this.inventoryService =
    this.client.getService<InventoryService>('InventoryService');
}
```

সহজভাবে: প্রথমে property ঘোষণা করা হচ্ছে, পরে gRPC service client দিয়ে সেটির মান দেওয়া হচ্ছে।

## ৬. Constructor এবং dependency injection

```ts
constructor(
  @Inject('INVENTORY_PACKAGE')
  private client: ClientGrpc,
) {}
```

এখানে NestJS dependency injection ব্যবহার করছে।

### `@Inject('INVENTORY_PACKAGE')`

`AppModule`-এ আগে লেখা ছিল:

```ts
ClientsModule.register([
  {
    name: 'INVENTORY_PACKAGE',
    transport: Transport.GRPC,
    // ...
  },
])
```

এই `name`-এর সঙ্গে `@Inject()`-এর token মিলতে হবে।

NestJS এই token ব্যবহার করে নিবন্ধিত gRPC client-টি constructor-এ inject করে।

### `private client: ClientGrpc`

- `client` হলো property-এর নাম।
- `ClientGrpc` হলো এর type।
- `private` লেখার কারণে TypeScript constructor parameter থেকে class property তৈরি করে দেয়।

এখন `this.client` ব্যবহার করে gRPC service access করা যাবে।

## ৭. `onModuleInit()` এবং `getService()`

```ts
onModuleInit() {
  this.inventoryService =
    this.client.getService<InventoryService>('InventoryService');
}
```

এই method-টি module initialization-এর সময় চলে।

`getService<InventoryService>('InventoryService')`-এর অর্থ:

- `getService()` — gRPC service-এর client proxy নেয়।
- `<InventoryService>` — TypeScript-কে জানায়, পাওয়া client-টিকে কোন interface-এর মতো ধরে ব্যবহার করতে হবে।
- `'InventoryService'` — `.proto` ফাইলে ঘোষিত service-এর নাম।

`.proto`-তে:

```proto
service InventoryService {
    rpc CheckStock (StockRequest) returns (StockResponse);
}
```

তাই service-এর নাম `'InventoryService'` দেওয়া হয়েছে।

এখন `this.inventoryService.checkStock()` ব্যবহার করা যাবে।

## ৮. `@Get('check-item')`

```ts
@Get('check-item')
async checkItem(@Query('pid') pid: string) {
```

এখানে একটি HTTP GET endpoint তৈরি হচ্ছে।

Controller-এ আছে:

```ts
@Controller('order')
```

তাই সম্পূর্ণ endpoint হবে:

```
GET /order/check-item?pid=123
```

### `@Query('pid') pid: string`

- `@Query('pid')` URL-এর query parameter থেকে `pid`-এর মান নেয়।
- `pid: string` হলো সেই মানের TypeScript type।
- `async` ব্যবহারের ফলে method-এর ভেতরে `await` ব্যবহার করা যায়।

যদি request হয়:

```
GET /order/check-item?pid=123
```

তাহলে:

```ts
pid === '123'
```

এখানে `pid`-এর মান string হিসেবে পাওয়া যায়।

## ৯. gRPC method call এবং `lastValueFrom()`

```ts
const stockStatus = await lastValueFrom(
  this.inventoryService.checkStock({ productId: pid })
);
```

এটাই কোডের সবচেয়ে গুরুত্বপূর্ণ অংশ।

ধাপে ধাপে কাজ:

```
১. HTTP request
   GET /order/check-item?pid=123
        │
        ▼
২. gRPC method call
   checkStock({ productId: pid })
        │
        ▼
৩. Observable → Promise
   lastValueFrom() এবং await
        │
        ▼
৪. stockStatus
   Inventory Service থেকে পাওয়া response
```

এখন প্রতিটি অংশ:

**`this.inventoryService.checkStock(...)`**

gRPC server-এ `CheckStock` RPC call পাঠায়। এখানে `pid`-এর মান `productId` হিসেবে পাঠানো হচ্ছে।

**`lastValueFrom(...)`**

gRPC method-এর Observable response-কে Promise-এ রূপান্তর করে।

**`await`**

Promise resolve হওয়া পর্যন্ত `async` function-এর পরবর্তী অংশ অপেক্ষা করে।

**`stockStatus`**

এতে gRPC server থেকে পাওয়া response রাখা হয়। উদাহরণ:

```ts
{
  availableQuantity: 50,
  inStock: true,
}
```

## ১০. `if (stockStatus.isStock)`

```ts
if (stockStatus.isStock) {
  return {
    status: 'Available',
    quantity: stockStatus.availableQuantity,
  };
}
```

এখানে response-এর stock status পরীক্ষা করা হচ্ছে।

- `stockStatus.isStock` সত্য হলে `Available` ফেরত দেওয়া হয়।
- `quantity`-তে পাওয়া stock quantity রাখা হয়।

কিন্তু আগের `.proto` এবং Inventory Controller-এ response field-এর নাম `inStock`, `isStock` নয়।

তাই এই কোডটি পরিবর্তন করতে হবে:

```ts
if (stockStatus.inStock) {
  return {
    status: 'Available',
    quantity: stockStatus.availableQuantity,
  };
}
```

এই নামটি `.proto` এবং server-এর response-এর সঙ্গে মিলিয়ে রাখতে হবে।

## ১১. Stock না থাকলে

```ts
return {
  status: 'Out of Stock',
  quantity: 0,
};
```

যদি `inStock` false হয়, তাহলে এই response ফেরত দেওয়া হবে:

```json
{
  "status": "Out of Stock",
  "quantity": 0
}
```

অর্থাৎ, stock নেই বা server-এর response অনুযায়ী stock শূন্য।

## ১২. সম্পূর্ণ flow-এর উদাহরণ

ধরা যাক, client এই request পাঠাল:

```
GET /order/check-item?pid=123
```

Inventory Service-এ product `123`-এর stock ৫০টি থাকলে response:

```json
{
  "status": "Available",
  "quantity": 50
}
```

আর product `456`-এর stock শূন্য হলে response:

```json
{
  "status": "Out of Stock",
  "quantity": 0
}
```

এই HTTP endpoint-এর JSON response তৈরি করছে Order Service-এর controller। Inventory Service শুধু stock-এর তথ্য ফেরত দিচ্ছে।

## সারকথা

`ClientGrpc` হলো gRPC client, `getService()` service proxy দেয়, `checkStock()` RPC call করে, আর `lastValueFrom()` ও `await` দিয়ে response পাওয়ার জন্য অপেক্ষা করা হয়।
