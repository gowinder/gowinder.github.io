# Class Constructor TypeOrmCrudService Cannot Be Invoked Without 'New'


# Class Constructor TypeOrmCrudService Cannot Be Invoked Without 'New' Error

## Cause

Recently I was developing a `chatgpt` gateway project using `nestjs`, with `nestjsx-crud` to speed up `curd` development, and I had the following code:

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

At runtime it threw the error:

```shell
Class Constructor TypeOrmCrudService Cannot Be Invoked Without 'New'
```

## Solution

I looked it up online and found one [here](https://github.com/nestjsx/crud/issues/378#issuecomment-764648149): `For anyone stumbling over this issue: it's probably a tsconfig issue. Try bumping your target from es5 to es6.`

Let me look at my project's `tsconfig.json`:

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

And compare it with a project that used to work:

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

Changing `target` to `es2017` fixed it:

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

