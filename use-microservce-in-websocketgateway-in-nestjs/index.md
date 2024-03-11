# nestjs整合websocketgateway和microservice



# nestjs整合websocketgateway和microservice

## 启因

使用`WebsocketGateway`做网关,接收客户端长连接后,将消息通过`RabbitMQ`消息队列转发到`microservice`,再做后续处理,`microservice`处理完成后,将结果也发送到`RabbitMQ`,`WebsocketGateway`接收到消息后,回传给客户端

### 目录结构

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

### 文件

#### 先看 `WSGateway`模块

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

注意:

- 在 `providers`中
  - 需要提供`WSGateway`,这样,`app.module`中才可以拿启动这个
  - 需要提供`SHARED_SERVICE`,这个是注入用的共享服务,没有可以不要
  - 需要提供`WS_QUEUE_SERVICE`, **这个很关键**, 没这个,`WSGateway`就只会启动`Websocket`监听,不会启动微服务
  - 上面这个好像不需要,去掉也能正常用,在`app.module`中创建就可以了

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

`gateway.controller.ts`, `gatewary.service.ts`与此无关,之前在里面也注入了`WSGateway`,结果所有的连接都被触发了两次,相当于启动了两个`WSGateway`服务.

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

- 直接创建微服务就可以了,所以在`ws-gateway.module.ts`中的微服务创建就不需要了
- 因为在`WSGateway.ts`中的`WebSocketGateway`的修饰器中已经定义了端口`3333`,所以`Websocket`也会直接启动,不需要再`app.listen(3333)`了

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

## 总结

到这里,`client`->`websocket`->`rabbitmq`->`app service`->`rabbitmq`->`websocket`->`client` 流程就通了.

中间走了些弯路,`nestjs`的文档感觉不是很全面和细致,然后我本身对`nestjs`的微服务也没有那么熟,所以花了些时间.

几个问题花了许多时间调试:

- 启动了两个`WSGateway`,所以一个连接会进来两次,这样我存的`clients`就不准,找不到对应的客户socket,就无法回复消息
- 怎么把`WSGateway`和`microservice`整合在一起,花了些时间,主要还是`nestjs`的模块架构,需要的时候要`import`,在本模块内要使用`imports`的模块中的`controller`等就需要在`providers`中加上,如果要给其它模块使用,就要在`exports`中加上.

