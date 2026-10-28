# JwtService Error in NestJS: Missing SECRET


# JwtService Error in NestJS: "JsonWebTokenError: secret or public key must be provided"

## Cause

This error keeps appearing recently: `JsonWebTokenError: secret or public key must be provided`,

- I have already added `imports: [JwtService]` in the environment variables,

- And I also added the following in the underlying library `shared.module.ts`:

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

But `JwtService` inside `AuthGurad` still can't find `secret`; based on past experience, it should be that `AuthModule` did not import it correctly.

---

## Solution

Replace `imports: [JwtService]` in `auth.module.ts` with `imports: [SharedModule]`.

Because the call to `JwtModule.registerAsync` is in `SharedModule`, and if you again directly `imports: [JwtModule]` in `auth.module.ts`, you may override the original one.

