# nestjs中,AuthGuard类,在CanActive函数中无法调用Inject.md


# nestjs中,AuthGuard类,在CanActive函数中无法调用Inject

## 启因

最近写`nestjs`项目,他的`UserGuard`很方便,相当于在`route`处理`request`时,可以经由`AuthGuard`来处理验证代码.

但是,我在写的过程中,出了一个问题,花了很长时间来解决,比如如下 `StaffAuthGuard.ts`代码:

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

在调用api时,经由`StaffAuthGuard.canActivate`时,就会报`undefined`,加了日志后,发现是这行:

```typescript
 this.authService
      .verifyJwt(jwt)
```

出错,此处就是`this.authService`未定义,但是编译可以通过,于是我在`constructor`函数中加上了日志,`this.authService`明明是有值的,为什么到调用的时候就没有了?

## 解决过程

百思不得其解,搞了几天,我都要放弃了,其间我还用了`ChatGPT`来帮我解决问题,他给了我的问题:

```
It's difficult to determine what could be causing this.authService to be undefined without seeing the entire code. However, here are a few common issues that could be causing the problem:

    The AuthService has not been imported correctly: Make sure that you have imported the AuthService using the correct import statement and path.

    The AuthService has not been added to the providers array in the module: If the AuthService is not part of the module's providers array, it won't be available for injection.

    The AuthService has not been injected into the constructor of the StaffJwtAuthGuard: If you are not injecting the AuthService into the constructor of the StaffJwtAuthGuard, then it won't be available as a property on the class.

Here's an example of how to correctly inject the AuthService into the StaffJwtAuthGuard:
```

照着对比了下,基本是这样做的,但是还是解决不了问题.

过了几天,中间我搞了点别的事情,换换心情,然后又回来解决这个,继续`google`,在`nestjs`的官方`issue`中找到一条: [# Nest can't resolve dependencies of the AuthGuard (Guard decorator) #893]([Nest can&#39;t resolve dependencies of the AuthGuard (Guard decorator) · Issue #893 · nestjs/nest · GitHub](https://github.com/nestjs/nest/issues/893)) 跟我的问题一样,也是在`AuthGuard`中无法引用`inject`后的服务,有一个回答: [Also ensure that you aren't providing AuthService twice]([Nest can&#39;t resolve dependencies of the AuthGuard (Guard decorator) · Issue #893 · nestjs/nest · GitHub](https://github.com/nestjs/nest/issues/893#issuecomment-406471157)) 引起了我的注意,

仔细观察日志,我发现`StaffAuthGuard`被初始化了两次,第一次日志里面有`authSerivce`,是正常有值的,第二次是`undefined`.顺着这个思路,把所有的`StaffAuthGuard`的地方都找出来,

使用`//`大法,一个个屏蔽测试,发现原因:

我的`StaffJwtAuth`是定义在`app`中,而需要用到此模块的`controller`在其引用的`shared`库中,按照官方文档(忘了具体位置了),如果在库中这样就需要在`app`中将`guard`定义为全局,如:

```typescript
 providers: [
    {
      provide: APP_GUARD,
      useClass: StaffJwtAuthGuard,
    },
  ],
```

我就是这样定义了全局了,但是,在我调用的api的`controller`位置,也加上了`@UseGuard(StaffJwtAuthGuard)`,就导致初始化了两次,并且第二次参数为`undefined`,将`controller`上的`@UseGuard`去掉后,就正常了.

---

## 思考

之后做了几个测试,把`StaffAuthModule`也放到库里面,屏掉全局`APP_GUARD`代码,在`controller`中加上代码,结果只有一个初始`constructor`调用,而且里面的`authService`是`undefined`
之后我又继续`google`, 在官方`issue`里面又找到一个: [middlewares are loosely coupled from modules, thus it may be better to 
use them instead. If you prefer to use guards, you need to guarantee an 
access to the `AuthService`. You can export this service from the `AuthModule`, and then import this module wherever you want to use your guard.](https://github.com/nestjs/nest/issues/244#issuecomment-346996421) ,意思就是说,在我这个场景中,在使用`StaffAuthGuard.StaffAuthService`的模块中,需要保证能访问到`StaffAuthService`, 于是,我在`controller`所在的模块的`imports`中增加了导入`StaffAuthModule`,结果,OK了,结论就是在使用`AuthGurad`的模块中,**因为`AuthGuard`是松散的,所有需要在调用他的模块中,保证他所有的依赖都会被import**

终于解决了,花了几天时间,不解决不舒服,现在舒服了,可以继续折腾了.

