# Integrating WebSocketGateway with Microservices in NestJS



# Integrating WebSocketGateway with Microservices in NestJS

## Motivation

Using `WebsocketGateway` as a gateway: after receiving the client's long-lived connection, forward the message to the `microservice` through the `RabbitMQ` message queue for further processing. After the `microservice` finishes processing, it also sends the result to `RabbitMQ`; when `WebsocketGateway` receives the message, it sends it back to the client.

### Directory structure

```text
src
  ws-gateway
    ws-gateway.controller.ts
    ws-gateway.module.ts
    ws-gateway.ts
  gateway.controller.ts
  gateway.module.ts
  gateway.service.ts
  main.ts
```

### Files

#### First, take a look at the `WSGateway` module

##### `WSGateway.ts`

```typescript
import { SHARED_SERVICE, SharedService } from '@app/shared';
import { Inject, Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import {
  ClientProxy,
  ClientProxyFactory,
  Ctx,
  MessagePattern,
  Payload,
  RmqContext,
  Server,
} from '@nestjs/microservices';
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  OnGatewayConnection,
  OnGatewayDisconnect,
  OnGatewayInit,
  ConnectedSocket,
} from '@nestjs/websockets';
import { SendMsgDTO } from './dto/send-msg.dto';
import { Socket } from '@nestjs/platform-socket.io';

@Injectable()
@WebSocketGateway(3333, {
  cors: {
    origin: '*',
  },
  namespace: 'main',
})
// implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
export class WSGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  apiClient: ClientProxy;
  clients: Map<string, Socket>;     // 存放客户端
  constructor(
    @Inject(SHARED_SERVICE)
    private readonly sharedService: SharedService,
    private readonly configService: ConfigService, 
  ) {
    console.log('WSGateway constructor: ');
    const client: ClientProxy = ClientProxyFactory.create(
      sharedService.getRmqOptions(configService.get('API_QUEUE'), ''),
    );

    this.apiClient = client;
    this.clients = new Map<string, Socket>();
  }
  @WebSocketServer() server: Server;

  afterInit(server: any) {
    console.log('websocket init:');
    this.server = server;
  }

  handleConnection(socket: Socket) {
    console.log('WebSocket client connecting, clients', this.clients.size);

    // 生成 socket_id，存储 socket 对象
    const socket_id = Math.random().toString(36).substring(7);

    socket['socket_id'] = socket_id;  //  将socket_id存到客户端socket连接对象中去
    this.clients.set(socket_id, socket);
  }

  handleDisconnect(socket: Socket) {
    console.log('WebSocket client disconnected');

    // 查找 socket_id，删除 socket 对象
    const socket_id = socket['socket_id'];
    console.log('remove socket:', socket_id);
    this.clients.delete(socket_id);
  }

  //  订阅客户端发来的send消息: socket.emit('send', data)
  @SubscribeMessage('send')
  async handleMessage(
    @ConnectedSocket()
    client: Socket,
    @MessageBody() data: SendMsgDTO,
  ): Promise<number> {
    console.log('; got [send]: prompt: ', data.prompt);

    //  数据中加入metadata, socket_id表示此消息属于哪个socket,回传的消息带上metadata就知道发给谁了
    const metadata = {
      socket_id: client['socket_id'],
      websocket_service: 'ws-gateway',
    };
    data['metadata'] = metadata;
    this.apiClient.emit({ cmd: 'send' }, data);
    return 0;
  }

  //  此为发送消息给客户端
  async emit(socket_id: string, event: any, content: string) {
    const client = this.clients.get(socket_id);
    const data = JSON.stringify(content);
    client.emit('recv', data);
  }
}

```

##### `ws-gateway.controller.ts`

```typescript
import { SHARED_SERVICE, SharedService } from '@app/shared';
import { Controller, Inject } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import {
  Ctx,
  MessagePattern,
  Payload,
  RmqContext,
} from '@nestjs/microservices';
import { WSGateway } from './ws-gateway';

@Controller()
export class WSGatewayController {
  constructor(
    @Inject(SHARED_SERVICE)
    private readonly sharedService: SharedService,
    private readonly gateway: WSGateway,
  ) {}

  //  订阅微服务事件,从rabbitmq读取 recv事件
  @MessagePattern({ cmd: 'recv' })
  async recv(@Ctx() context: RmqContext, @Payload() content: string) {
    console.log('GatewayController got [recv]: ', content);
    this.sharedService.acknowledgeMessage(context);

    //  调用 wsgateway发送
    return this.gateway.emit(content['metadata']['socket_id'], 'recv', content);
  }
}
```

##### `ws-gateway.module.ts`

Notes:

- In `providers`
  - You need to provide `WSGateway` so that it can be started in `app.module`
  - You need to provide `SHARED_SERVICE`, which is the shared service used for injection; you can leave it out if you don't have one
  - You need to provide `WS_QUEUE_SERVICE`, **this is critical**; without it, `WSGateway` will only start the `Websocket` listener and will not start the microservice
  - The above one seems unnecessary; it works fine after removing it, and creating it in `app.module` is enough

```typescript
import { SharedModule, SharedService, SHARED_SERVICE } from '@app/shared';
import { Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { ClientProxyFactory } from '@nestjs/microservices';
import { WSGateway } from './ws-gateway';
import { WSGatewayController } from './ws-gateway.controller';

@Module({
  imports: [SharedModule],
  controllers: [WSGatewayController],
  providers: [
    WSGateway,
    {
      provide: SHARED_SERVICE,
      useClass: SharedService,
    },
    {
      provide: 'WS_QUEUE_SERVICE',
      useFactory: (
        sharedService: SharedService,
        configSerivce: ConfigService,
      ) => {
        const rqmOptions = sharedService.getRmqOptions(
          configSerivce.get('WS_QUEUE'),
          '',
        );
        return ClientProxyFactory.create(rqmOptions);
      },
      inject: [SharedService, ConfigService],
    },
  ],
  exports: ['WS_QUEUE_SERVICE', WSGateway],
})
export class WSGatewayModule {}

```

#### app

`gateway.controller.ts` and `gatewary.service.ts` are unrelated to this. Previously `WSGateway` was also injected there, and as a result every connection was triggered twice — effectively starting two `WSGateway` services.

##### `gateway.module.ts`

```typescript
import { SharedModule, SharedService, SHARED_SERVICE } from '@app/shared';
import { Module } from '@nestjs/common';
import { GatewayService } from './gateway.service';
import { WSGatewayController } from './ws-gateway/ws-gateway.controller';
import { WSGatewayModule } from './ws-gateway/ws-gateway.module';

@Module({
  imports: [WSGatewayModule, SharedModule],
  controllers: [WSGatewayController],
  providers: [
    GatewayService,
    {
      provide: SHARED_SERVICE,
      useClass: SharedService,
    },
  ],
})
export class GatewayModule {}
```

#### main.ts

- You can just create the microservice directly, so creating the microservice in `ws-gateway.module.ts` is no longer needed
- Because the port `3333` is already defined in the `WebSocketGateway` decorator in `WSGateway.ts`, `Websocket` will also start directly; there is no need to call `app.listen(3333)`

```typescript
import { SharedModule, SharedService } from '@app/shared';
import { ConfigService } from '@nestjs/config';
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions } from '@nestjs/microservices';
import { GatewayModule } from './gateway.module';

async function bootstrap() {
  const configApp = await NestFactory.create(SharedModule);
  const configService = configApp.get(ConfigService);
  const sharedService = configApp.get(SharedService);

  const microserviceApp =
    await NestFactory.createMicroservice<MicroserviceOptions>(
      GatewayModule,
      sharedService.getRmqOptions(configService.get('WS_QUEUE'), ''),
    );
  await microserviceApp.listen();
}
bootstrap();
```

## Summary

At this point, the `client` -> `websocket` -> `rabbitmq` -> `app service` -> `rabbitmq` -> `websocket` -> `client` flow works.

I took some detours along the way. NestJS's documentation doesn't feel very comprehensive or detailed, and I'm not that familiar with NestJS microservices myself, so it took some time.

A few problems took a lot of time to debug:

- Two `WSGateway` instances were started, so a connection came in twice; this made the `clients` I stored inaccurate, and I couldn't find the corresponding client socket, so I couldn't reply to messages
- Figuring out how to integrate `WSGateway` with the `microservice` took some time. The main issue is NestJS's module architecture: you need to `import` when necessary, and to use a `controller` from an imported module within the current module you need to add it to `providers`; if you want other modules to use it, you need to add it to `exports`.

