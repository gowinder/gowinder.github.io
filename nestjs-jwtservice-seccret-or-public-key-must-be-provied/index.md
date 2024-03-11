# nestjs中,JwtService报错,没有SECRET


# nestjs中,JwtService报错: "JsonWebTokenError: secret or public key must be provided"

## 原因

最近老是出现这个错误: `JsonWebTokenError: secret or public key must be provided`, 

- 我已经在环境变量中增加`imports: [JwtService]`,

- 也在底层库`shared.module.ts`里面增加了:
  
  ```typescript
    imports: [
      JwtModule.registerAsync({
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => ({
          secret: configService.get<string>('JWT_SECRET') as string,
          signOptions: { expiresIn: '1d' },
        }),
        inject: [ConfigService],
      }),
  ]
  ```

结果`AuthGurad`中的`JwtService`还是找不到`secret`, 按照之前的经验,应该是`AuthModule`中没有正确导入

---

## 解决办法

将 `auth.module.ts`中的`imports: [JwtService]`换成`imports: [SharedModule]`

因为调用`JwtModule.registerAsync`是在`SharedModule`中,而如果在`auth.module.ts`中再次直接`imports: [JwtModule]`,可能将原来的覆盖了.

