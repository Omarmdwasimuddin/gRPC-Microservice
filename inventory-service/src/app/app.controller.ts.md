# NestJS gRPC `CheckStock` কোড ব্যাখ্যা

এখানে মূলত `InventoryService`-এর `CheckStock` RPC method-এর request গ্রহণ করা, stock পরীক্ষা করা এবং response ফেরত দেওয়ার কাজ হচ্ছে।

## ১. `GrpcMethod` import

```ts
import { GrpcMethod } from '@nestjs/microservices';
```

- `@nestjs/microservices` হলো NestJS-এর microservices প্যাকেজ।
- `GrpcMethod` হলো একটি decorator, যা কোনো NestJS method-কে নির্দিষ্ট gRPC service-এর RPC method-এর সঙ্গে যুক্ত করে।
- এর মাধ্যমে gRPC client থেকে আসা request-কে সঠিক method-এ পাঠানো যায়।

সহজভাবে: gRPC client যখন `CheckStock` কল করবে, তখন NestJS যেন বুঝতে পারে কোন function চালাতে হবে, `GrpcMethod` সেটি নির্ধারণ করতে সাহায্য করে।

## ২. `@GrpcMethod('InventoryService', 'CheckStock')`

```ts
@GrpcMethod('InventoryService', 'CheckStock')
```

এখানে দুটি argument রয়েছে।

| Argument             | কাজ                                    |
| -------------------- | -------------------------------------- |
| `'InventoryService'` | `.proto` ফাইলে ঘোষিত service-এর নাম    |
| `'CheckStock'`       | `.proto` ফাইলে ঘোষিত RPC method-এর নাম |

`.proto` ফাইলে ছিল:

```proto
service InventoryService {
    rpc CheckStock (StockRequest) returns (StockResponse);
}
```

এই দুই জায়গার নাম মিলতে হবে।

যখন client `InventoryService`-এর `CheckStock` method কল করবে, NestJS নিচের `checkStock()` function-টি চালাবে।

> মনে রাখবে: `CheckStock` হলো gRPC method-এর নাম, আর `checkStock` হলো TypeScript function-এর নাম। দুটি নাম আলাদা হলেও সমস্যা নেই।

## ৩. `checkStock(data: { productId: string })`

```ts
checkStock(data: { productId: string }) {
```

এটি সেই function, যা gRPC request গ্রহণ করে।

- `checkStock` — function-এর নাম।
- `data` — client থেকে আসা request-এর data।
- `{ productId: string }` — TypeScript-এর object type; এতে `productId` নামে একটি string property থাকবে।

ধরা যাক, client এই request পাঠাল:

```json
{
  "productId": "123"
}
```

তাহলে function-এর মধ্যে:

```ts
data.productId // "123"
```

পাওয়া যাবে।

এখানে `data` নামটি নিজের ইচ্ছামতো দেওয়া যায়। যেমন `request` দিলেও কাজ করবে, যদি function-এর ভেতরেও সেই নাম ব্যবহার করা হয়।

## ৪. `const items: Record<string, number>`

```ts
const items: Record<string, number> = {
  '123': 50,
  '456': 0,
};
```

এখানে আপাতত একটি সাধারণ JavaScript object-কে stock-এর ডেটা হিসেবে ব্যবহার করা হয়েছে।

- `const items` — `items` নামে একটি variable তৈরি করেছে।
- `Record<string, number>` — TypeScript-এর একটি type, যেখানে key হবে `string` এবং value হবে `number`।
- `'123': 50` — product ID `123`-এর stock ৫০টি।
- `'456': 0` — product ID `456`-এর stock শূন্য।

এটি একটি in-memory data structure, আসল database নয়।

## ৫. `const qty = items[data.productId] || 0;`

```ts
const qty = items[data.productId] || 0;
```

এই লাইনটি request-এর product ID দিয়ে stock-এর পরিমাণ খুঁজে বের করে।

ধাপে ধাপে:

1. `data.productId` দিয়ে client-এর পাঠানো ID পাওয়া যায়।
2. `items[data.productId]` দিয়ে `items` object থেকে সংশ্লিষ্ট stock বের করা হয়।
3. `|| 0` দিয়ে আগের ফলাফল falsy হলে `0` নেওয়া হয়।

উদাহরণ:

| `productId` | `items[data.productId]` | `qty` |
| ----------- | ----------------------- | ----- |
| `"123"`     | `50`                    | `50`  |
| `"456"`     | `0`                     | `0`   |
| `"999"`     | `undefined`             | `0`   |

এখানে একটি গুরুত্বপূর্ণ বিষয়: JavaScript-এ `0` একটি falsy value। তাই stock শূন্য হলেও `qty` হবে `0`।

## ৬. `return` দিয়ে response পাঠানো

```ts
return {
  availableQuantity: qty,
  isStock: qty > 0,
};
```

এই object-টি gRPC response হিসেবে ফেরত দেওয়া হবে।

- `availableQuantity: qty` — পণ্যের মজুতের পরিমাণ ফেরত দেয়।
- `isStock: qty > 0` — stock শূন্যের বেশি হলে `true`, অন্যথায় `false`।

উদাহরণস্বরূপ, `productId: "123"` হলে:

```json
{
  "availableQuantity": 50,
  "isStock": true
}
```

তবে আগের `.proto` ফাইলে response field-এর নাম ছিল `inStock`, `isStock` নয়।

তাই সঠিক কোড হবে:

```ts
return {
  availableQuantity: qty,
  inStock: qty > 0,
};
```

কারণ `.proto` ফাইলে:

```proto
message StockResponse {
    int32 availableQuantity = 1;
    bool inStock = 2;
}
```

## সারকথা

`@GrpcMethod` request-কে সঠিক handler-এর সঙ্গে যুক্ত করে, `data` থেকে product ID পাওয়া যায়, `items` থেকে stock খোঁজা হয়, আর `return` দিয়ে stock-এর তথ্য client-কে পাঠানো হয়।
