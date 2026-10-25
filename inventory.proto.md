# gRPC Inventory Service — Protocol Buffers

## 1. সংশোধিত `inventory.proto` কোড

তোমার মূল কোডটি প্রায় সঠিক ছিল। তবে `CheckStock` RPC method-এর response type হিসেবে `StockRequest` না দিয়ে `StockResponse` ব্যবহার করতে হবে।

```protobuf
syntax = "proto3";

package inventory;

service InventoryService {
    rpc CheckStock (StockRequest) returns (StockResponse);
}

message StockRequest {
    string productId = 1;
}

message StockResponse {
    int32 availableQuantity = 1;
    bool inStock = 2;
}
```

এই `.proto` ফাইলটি gRPC service-এর **contract** নির্ধারণ করে—ক্লায়েন্ট কী পাঠাবে এবং সার্ভার কী ধরনের response দেবে।

---

## 2. লাইন বাই লাইন ব্যাখ্যা

### `syntax = "proto3";`

```protobuf
syntax = "proto3";
```

- `syntax` দিয়ে Protocol Buffers-এর syntax version নির্ধারণ করা হয়।
- `"proto3"` হলো Protocol Buffers-এর তৃতীয় সংস্করণের syntax।
- `.proto` ফাইলে service, request ও response-এর structure নির্ধারণ করা যায়।

**সহজভাবে:** `.proto` ফাইল কোন syntax version অনুসরণ করবে, এই লাইন সেটি নির্ধারণ করে।

### `package inventory;`

```protobuf
package inventory;
```

- `package` service ও message-গুলোর namespace নির্ধারণ করে।
- এখানে `inventory` হলো package-এর নাম।
- বড় প্রজেক্টে একই নামের service বা message-এর মধ্যে সংঘর্ষ এড়াতে এটি সাহায্য করে।

### `service InventoryService`

```protobuf
service InventoryService {
    rpc CheckStock (StockRequest) returns (StockResponse);
}
```

#### `service InventoryService`

একটি gRPC service ঘোষণা করে, যার নাম `InventoryService`।

#### `rpc CheckStock`

- `rpc`-এর পূর্ণরূপ **Remote Procedure Call**।
- `CheckStock` হলো method-এর নাম।
- অন্য একটি সার্ভিস এই method কল করে কোনো পণ্যের stock সম্পর্কে জানতে চাইতে পারে।

#### `(StockRequest) returns (StockResponse)`

| অংশ | কাজ |
|---|---|
| `StockRequest` | ক্লায়েন্ট যে request পাঠাবে |
| `CheckStock` | যে RPC method কল হবে |
| `StockResponse` | সার্ভার যে ধরনের response ফেরত দেবে |

অর্থাৎ, ক্লায়েন্ট পণ্যের ID পাঠাবে এবং সার্ভার stock-এর তথ্য ফেরত দেবে।

### `message StockRequest`

```protobuf
message StockRequest {
    string productId = 1;
}
```

এটি request-এর data structure নির্ধারণ করে।

- `message StockRequest` — request message-এর নাম।
- `string` — `productId`-এর data type; এটি text হিসেবে ব্যবহৃত হবে।
- `productId` — পণ্যের ID রাখবে।
- `= 1` — field number; এটি পণ্যের ID-এর মান `1` বোঝায় না।

উদাহরণ হিসেবে request-এর তথ্য এমন হতে পারে:

```json
{
  "productId": "PROD-101"
}
```

### `message StockResponse`

```protobuf
message StockResponse {
    int32 availableQuantity = 1;
    bool inStock = 2;
}
```

এটি response-এর data structure নির্ধারণ করে।

#### `int32 availableQuantity = 1;`

- `int32` হলো 32-bit signed integer।
- `availableQuantity` পণ্যের মজুতের পরিমাণ প্রকাশ করবে।
- `= 1` হলো field number।

উদাহরণ: `25`

#### `bool inStock = 2;`

- `bool` হলো Boolean data type।
- `inStock` পণ্য মজুত আছে কি না বোঝাবে।
- `= 2` হলো field number।
- এর মান `true` অথবা `false` হতে পারে।

উদাহরণস্বরূপ response:

```json
{
  "availableQuantity": 25,
  "inStock": true
}
```

এটি বোঝায়, পণ্যটির ২৫টি ইউনিট মজুত রয়েছে।

---

## 3. gRPC communication কীভাবে কাজ করে?

```text
Order Service (Client)
        |
        | productId পাঠায়
        v
gRPC + inventory.proto
        |
        | Request/Response contract অনুসরণ করে
        v
Inventory Service (Server)
        |
        | Stock পরীক্ষা করে
        v
StockResponse
(availableQuantity, inStock)
```

উদাহরণ: `Order Service` অর্ডার তৈরি করার আগে `InventoryService`-কে জিজ্ঞাসা করতে পারে পণ্যটি মজুত আছে কি না। Inventory Service ডেটাবেস থেকে stock পরীক্ষা করে response পাঠাবে।

---

## 4. গুরুত্বপূর্ণ বিষয়

- `.proto` ফাইল communication contract নির্ধারণ করে।
- `.proto` ফাইল নিজে ডেটাবেস থেকে stock পরীক্ষা করে না।
- stock পরীক্ষা করার business logic NestJS server-side implementation-এ লিখতে হবে।
- `StockRequest` হলো request-এর structure; `StockResponse` হলো response-এর structure।
- `= 1` ও `= 2` হলো Protocol Buffers field number, field value নয়।
- `CheckStock` এখানে একটি unary RPC: একটি request পাঠানো হয় এবং একটি response ফেরত আসে।

## 5. পরবর্তী ধাপ

পরবর্তী ধাপে NestJS-এ `InventoryService` implement করতে হবে এবং `CheckStock` RPC method-এর handler তৈরি করতে হবে।
