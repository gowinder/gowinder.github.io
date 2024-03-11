# Class Constructor TypeOrmCrudService Cannot Be Invoked Without 'New'


# Class Constructor TypeOrmCrudService Cannot Be Invoked Without 'New' 错误

## 启因

最近开发一个`chatgpt`网关项目,使用`nestjs`,其中用`nestjsx-crud`来提高`curd`开发效率,有以下代码:

```typescript
@Injectable()
export class ChatgptAccountService extends TypeOrmCrudService<ChatGPTAccount> {
  constructor(
    @InjectRepository(ChatGPTAccount)
    private readonly chatgptAccountRepository: Repository<ChatGPTAccount>,
  ) {
    super(chatgptAccountRepository);
  }
```

运行时会报错:

```shell
Class Constructor TypeOrmCrudService Cannot Be Invoked Without 'New'
```

## 解决

网上查了下,在[这里](https://github.com/nestjsx/crud/issues/378#issuecomment-764648149)找到一个: `For anyone stumbling over this issue: it's probably a tsconfig issue. Try bumping your target from es5 to es6.`

看一下我的项目的`tsconfig.json`:

```json
{
   "compilerOptions": {
      "lib": [
         "es5",
         "es6"
      ],
      "target": "es5",
      "module": "commonjs",
      "moduleResolution": "node",
      "outDir": "./build",
      "emitDecoratorMetadata": true,
      "experimentalDecorators": true,
      "sourceMap": true
   }
}
```

再对比一下之前正常的项目:

```json
"compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "es2017",
    "sourceMap": true,
    "outDir": "./dist",
    ...
```

把`target`改成: `es2017`就好了:

```shell
{
  "compilerOptions": {
    "target": "es2017",
    "module": "commonjs",
    "moduleResolution": "node",
    "outDir": "./build",
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "sourceMap": true
  }
}
```

