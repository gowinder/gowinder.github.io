# Mutual References Between Entities in NestJS and TypeORM


# Mutual References Between Entities in NestJS and TypeORM

`20230131-152600`

When using `nestjs` with `typeorm`, entities that reference each other keep throwing errors; following the [official docs](https://typeorm.io/relations-faq#avoid-circular-import-errors), I always ended up with an `entity metadata not found` error.

## The approach that fails

After many attempts, I found it needs to be written like this. Suppose there are two `entity`s, one `UserEntity` and one `CarEntity`, referencing each other. Writing them the following way throws `circular import errors`.

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

This way it reports a circular reference error: `circular import errors`,

## Writing everything the official way also fails

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

This way it reports the `entity metadata not found` error again.

## The correct approach

The correct approach is to use `import type` in only one file, and keep the normal `import` in the other.

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

This way it's OK.

tag: nestjs, typeorm, entity, metadata not found, circular import errors

