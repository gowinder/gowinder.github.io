# Nestjs, typeorm中的entity中的互相引用问题


# Nestjs, typeorm中的entity中的互相引用问题

`20230131-152600`

使用`nestjs`时,用到`typeorm`,有互相引用关系的`entity`,老是报错,按照[官方文档](https://typeorm.io/relations-faq#avoid-circular-import-errors),最后总会报一个`entity metadata not found`的错误.

## 会出错的方式

试了好多次,发现需要这样写,假设有两个`entity`, 一个是`UserEntity`, 一个是`CarEntity`, 互相引用,按以下方法写,会报`circular import errors`

### user.entity.ts

```javascript
import {
  Column,
  Entity,
  ManyToMany,
  PrimaryGeneratedColumn,
} from 'typeorm';
import { CarEntity } from './car.entity.ts'

@Entity({ name: 'users' })
export class UserEntity() {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany(() => CarEntity, (car) => car.id)
  car: CarEntity
}
```

### car.entity.ts

```javascript
import {
  Column,
  Entity,
  ManyToMany,
  PrimaryGeneratedColumn,
} from 'typeorm';
import { UserEntity } from './user.entity.ts'

@Entity({ name: 'users' })
export class CarEntity() {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany(() => UserEntity, (user) => user.id)
  user: UserEntity
}
```

这种方式就会报找不到循环引用错误的错误: `circular import errors`,

## 全部按官方方式写,也会报错

### user.entity.ts

```javascript
import {
  Column,
  Entity,
  ManyToMany,
  PrimaryGeneratedColumn,
} from 'typeorm';
import type { CarEntity } from './car.entity.ts'

@Entity({ name: 'users' })
export class UserEntity() {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany('CarEntity', 'id')
  car: CarEntity
}
```

### car.entity.ts

```javascript
import {
  Column,
  Entity,
  ManyToMany,
  PrimaryGeneratedColumn,
} from 'typeorm';
import type { UserEntity } from './user.entity.ts'

@Entity({ name: 'users' })
export class CarEntity() {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany('UserEntity', 'id')
  user: UserEntity
}
```

这样的方式又会报`entity metadata not found`的错误.

## 正确的方式

正确的方式是,只需要一个文件按`import type`方式,另一个文件还是正常`import`

user.entity.ts

```javascript
import {
  Column,
  Entity,
  ManyToMany,
  PrimaryGeneratedColumn,
} from 'typeorm';
import { CarEntity } from './car.entity.ts'

@Entity({ name: 'users' })
export class UserEntity() {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany(() => CarEntity, (car) => car.id)
  car: CarEntity
}
```

### car.entity.ts

```javascript
import {
  Column,
  Entity,
  ManyToMany,
  PrimaryGeneratedColumn,
} from 'typeorm';
import type { UserEntity } from './user.entity.ts'

@Entity({ name: 'users' })
export class CarEntity() {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany('UserEntity', 'id')
  user: UserEntity
}
```

这样就OK了.

tag: nestjs, typeorm, entity, metadata not found, circular import errors

