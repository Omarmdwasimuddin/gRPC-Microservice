## gRPC Microservices with NestJS and Nx


#### Create a NestJS Nx Workspace
```bash
npx create-nx-workspace@latest wasuTech-gRPC --preset=nest
```
> Enter press koro
<img width="777" height="244" alt="image" src="https://github.com/user-attachments/assets/b6890e90-8a64-4ad3-b462-ae76bb39f60f" />

<img width="710" height="283" alt="image" src="https://github.com/user-attachments/assets/dd54f440-0136-4ee9-96cd-cbd13706e3ef" />

<img width="687" height="335" alt="image" src="https://github.com/user-attachments/assets/a19ab57e-c7b4-4ce4-9416-eaddfb8d78dc" />

<img width="702" height="392" alt="image" src="https://github.com/user-attachments/assets/c5222058-dee7-4fa2-895c-a6f9664a54a1" />

<img width="674" height="450" alt="image" src="https://github.com/user-attachments/assets/efd6c5ab-fc91-457e-ab1a-77f5948b7ad0" />

<img width="773" height="570" alt="image" src="https://github.com/user-attachments/assets/2dc33906-319c-4b48-8f6c-ee2785310484" />

<img width="720" height="620" alt="image" src="https://github.com/user-attachments/assets/1f526712-3da5-402c-b1e2-f327af988012" />

---

#### Navigate to the Project Directory
```bash
cd wasuTech-gRPC
```
---
#### Generate the Inventory Service
```bash 
npx nx generate @nx/nest:app inventory-service
```
<img width="785" height="296" alt="image" src="https://github.com/user-attachments/assets/06649baa-11a6-404f-89d0-ab0d503c6e32" />

<img width="783" height="327" alt="image" src="https://github.com/user-attachments/assets/6996e807-6c6b-456b-9afc-4d7b2c035952" />

---

#### Generate the Order Service
```bash
npx nx generate @nx/nest:app order-service
```
---


####
```bash
npm i @grpc/grpc-js @grpc/proto-loader "@nestjs/microservices@^11"
```
---

#### root directory te---> `libs/proto/inventory.proto`
```ts
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
---

#### `inventory-service/src/main.ts`
```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app/app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
    transport: Transport.GRPC,
    options: {
      package: 'inventory',
      protoPath: join(process.cwd(), 'libs/proto/inventory.proto'),
      url: 'localhost:50051', 
    }
  });

  await app.listen();
  console.log('Inventory gRPC service is running!')
}

bootstrap();
```
---

#### `inventory-service/src/app/app.controller.ts`
```ts
import { Controller } from '@nestjs/common';
import { GrpcMethod } from '@nestjs/microservices';

@Controller()
export class AppController {
  @GrpcMethod('InventoryService', 'CheckStock')
  checkStock(data: {productId: string}) {
    const items: Record<string, number> = {
      '123': 50, 
      '456': 0,
    }
    const qty = items[data.productId] || 0;
    return { 
      availableQuantity: qty,
      inStock: qty > 0, 
    };
  }
}
```
---

#### `order-service/src/app/app.module.ts`
```ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { AppController } from './app.controller';
import { join } from 'path';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'INVENTORY_PACKAGE',
        transport: Transport.GRPC,
        options: {
          package: 'inventory',
          protoPath: join(process.cwd(), 'libs/proto/inventory.proto'),
          url: 'localhost:50051',
        }
      }
    ])
  ],
  controllers: [AppController],
})
export class AppModule {}
```
---

#### `order-service/src/app/app.controller.ts`
```ts
import { Controller, Get, Inject, OnModuleInit, Query, } from '@nestjs/common';
import { Observable, lastValueFrom } from 'rxjs';
import { ClientGrpc } from '@nestjs/microservices';

interface InventoryService {
  checkStock(data: { productId: string }) : Observable<any>;
}

@Controller('order')
export class AppController implements OnModuleInit {
  private inventoryService!: InventoryService;

  constructor(@Inject('INVENTORY_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    this.inventoryService = this.client.getService<InventoryService>('InventoryService');
  }

  @Get('check-item')
  async checkItem(@Query('pid') pid: string) {
    const stockStatus = await lastValueFrom(
      this.inventoryService.checkStock({ productId: pid })
    )
    
    if(stockStatus.inStock) {
      return {
        status: 'Available',
        quantity: stockStatus.availableQuantity
      }
    }
    return {
      status: 'Out of Stock',
      quantity: 0,
    }
  }
}
```
---

####
```bash
npx nx serve inventory-service
```
---

####
```bash
npx nx serve order-service
```
---

#### Browser e hit koro
```bash
localhost:3000/api/order/check-item?pid=123
```
```bash
localhost:3000/api/order/check-item?pid=456
```
---

#### OUTPUT
<img width="485" height="157" alt="image" src="https://github.com/user-attachments/assets/cb6b7048-bf85-49c4-ae82-9417e7c0789f" />

<img width="494" height="172" alt="image" src="https://github.com/user-attachments/assets/fc12535a-f2c6-464b-a1fe-f45f378882ae" />
