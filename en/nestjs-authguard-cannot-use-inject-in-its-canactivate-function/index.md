# NestJS: AuthGuard Cannot Use Inject in Its canActivate Function


# NestJS: AuthGuard Cannot Use Inject in Its canActivate Function

## Cause

Recently I was writing a `nestjs` project. Its `UserGuard` is very convenient — it is like, when a `route` handles a `request`, the verification code can be handled by `AuthGuard`.

But while writing it, I ran into a problem that took me a very long time to solve, for example the following `StaffAuthGuard.ts` code:

```typescript
import { ExecutionContext, Inject, Injectable } from '@nestjs/common';
import { StaffAuthService } from './staff-auth.service';
import { of } from 'rxjs';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class StaffJwtAuthGuard extends AuthGuard('jwt') {
  constructor(private authService: StaffAuthService) {
    super(authService);
    this.authService = authService;
    console.log(
      '🚀 ~ file: staff-auth.guard.ts:11 ~ StaffJwtAuthGuard ~ constructor ~ authService',
      authService,
      ', this.authService: ',
      this.authService,
    );
  }
  canActivate(context: ExecutionContext) {
    if (context.getType() != 'http') {
      return false;
    }
    const authHeader = context.switchToHttp().getRequest()
      .headers.authorization;
    if (!authHeader) {
      return false;
    }
    const authHeaderParts = authHeader.split(' ');
    if (authHeaderParts.length != 2) {
      return false;
    }
    const [, jwt] = authHeaderParts;

    this.authService
      .verifyJwt(jwt)
      .then((res) => {
        const exp = res.exp;
        if (!exp) return of(false);
        const TOKEN_EXP_MS = exp * 1000;
        const isJwtValid = Date.now() < TOKEN_EXP_MS;
        // return of(isJwtValid);
        return isJwtValid;
      })
      .catch((err) => {
        console.log(
          '🚀 ~ file: staff-auth.guard.ts:35 ~ StaffJwtAuthGuard ~ canActivate ~ err',
          err,
        );
        // return of(false);
        return false;
      });
    return super.canActivate(context);
  }
}
```

When calling the api, when it went through `StaffAuthGuard.canActivate`, it reported `undefined`. After adding logs, I found it was this line:

```typescript
 this.authService
      .verifyJwt(jwt)
```

that errored — here `this.authService` is undefined, yet it compiles fine. So I added logs in the `constructor` function; `this.authService` clearly has a value, so why is it gone by the time it is called?

## Solution process

I couldn't figure it out and struggled for several days; I was about to give up. In between I also used `ChatGPT` to help me solve the problem, and this is what it gave me:

```
It's difficult to determine what could be causing this.authService to be undefined without seeing the entire code. However, here are a few common issues that could be causing the problem:

    The AuthService has not been imported correctly: Make sure that you have imported the AuthService using the correct import statement and path.

    The AuthService has not been added to the providers array in the module: If the AuthService is not part of the module's providers array, it won't be available for injection.

    The AuthService has not been injected into the constructor of the StaffJwtAuthGuard: If you are not injecting the AuthService into the constructor of the StaffJwtAuthGuard, then it won't be available as a property on the class.

Here's an example of how to correctly inject the AuthService into the StaffJwtAuthGuard:
```

Following that comparison, that is basically what I did, but it still didn't solve the problem.

A few days later, in between I did something else to change my mood, then came back to solve this, continued to `google`, and found an issue in `nestjs`'s official `issue` list: [# Nest can't resolve dependencies of the AuthGuard (Guard decorator) #893]([Nest can&#39;t resolve dependencies of the AuthGuard (Guard decorator) · Issue #893 · nestjs/nest · GitHub](https://github.com/nestjs/nest/issues/893)) — the same as my problem, also unable to reference the `inject`ed service in `AuthGuard`. One answer caught my attention: [Also ensure that you aren't providing AuthService twice]([Nest can&#39;t resolve dependencies of the AuthGuard (Guard decorator) · Issue #893 · nestjs/nest · GitHub](https://github.com/nestjs/nest/issues/893#issuecomment-406471157)),

Observing the logs carefully, I found that `StaffAuthGuard` was initialized twice. The first log had `authSerivce` with a normal value, the second was `undefined`. Following this line of thought, I found all the places with `StaffAuthGuard`,

using the `//` method, shielding them one by one for testing, and found the cause:

My `StaffJwtAuth` is defined in the `app`, while the `controller` that needs to use this module is in the `shared` library it references. According to the official docs (I forgot the exact location), if it is in a library like this, you need to define the `guard` as global in the `app`, such as:

```typescript
 providers: [
    {
      provide: APP_GUARD,
      useClass: StaffJwtAuthGuard,
    },
  ],
```

That is how I defined it globally, but at the `controller` of the api I was calling, I also added `@UseGuard(StaffJwtAuthGuard)`, which caused it to be initialized twice, and the second time the parameter was `undefined`. After removing `@UseGuard` on the `controller`, it worked normally.

---

## Thoughts

Afterwards I did a few tests, also putting `StaffAuthModule` into the library, shielding the global `APP_GUARD` code, and adding code in the `controller`. The result was only one initial `constructor` call, and the `authService` inside was `undefined`.
Then I continued to `google`, and found another one in the official `issue`: [middlewares are loosely coupled from modules, thus it may be better to 
use them instead. If you prefer to use guards, you need to guarantee an 
access to the `AuthService`. You can export this service from the `AuthModule`, and then import this module wherever you want to use your guard.](https://github.com/nestjs/nest/issues/244#issuecomment-346996421), meaning that in my scenario, in the module that uses `StaffAuthGuard.StaffAuthService`, I need to ensure it can access `StaffAuthService`. So I added the import of `StaffAuthModule` to the `imports` of the module where the `controller` is located, and the result was OK. The conclusion is that in the module that uses `AuthGuard`, **because `AuthGuard` is loosely coupled, you need to ensure all its dependencies are imported in the module that calls it**

Finally solved. It took several days, and I'm not comfortable unless I solve it; now I'm comfortable, so I can keep tinkering.

