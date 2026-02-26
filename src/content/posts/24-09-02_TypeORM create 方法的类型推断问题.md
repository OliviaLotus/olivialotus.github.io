---
title: TypeORM create方法的类型推断问题
published: 2024-09-02
description: TypeORM create方法的类型推断问题
tags: [nodejs, typeorm]
category: 前端
---

## 问题描述

今天从AI那里复制了一段代码,逻辑看起来没有问题

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';
import * as bcrypt from 'bcryptjs';

@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
  ) {}

  async createUser(userData: any): Promise<User> {
    const hashedPassword = bcrypt.hashSync(userData.password, 10);
    const user = this.userRepository.create({ ...userData, password: hashedPassword });
    return this.userRepository.save(user);
  }
}
```

然而，VSCode 或 TypeScript 编译器可能会报错，提示 `create` 方法返回的是一个数组，而不是单个 `User` 对象。**有点疑惑**传个对象，推出数组来了。

## 问题发现

### 1. `TypeORM` 的 `create` 方法重载

`TypeORM` 的 `create` 方法有多个重载版本：

```ts
create<Entity>(entityLike: DeepPartial<Entity>): Entity;
create<Entity>(entityLikes: DeepPartial<Entity>[]): Entity[];
```

- 如果你传入一个对象（`DeepPartial<Entity>`），它会返回一个实体对象（`Entity`）。  
- 如果你传入一个对象数组（`DeepPartial<Entity>[]`），它会返回一个实体对象数组（`Entity[]`）。

### 2. 为什么 TypeScript 会推断为数组？

TypeScript 的类型推断机制会根据上下文和传入的参数来选择最合适的方法重载版本,但说实话我搞不懂他的逻辑,但是推测如下：

1. **类型不明确**  
   如果 `userData` 的类型不够明确，TypeScript 可能会误判为数组。例如，如果 `userData` 的类型是 `any` 或者是一个复杂的类型，TypeScript 可能会无法准确推断其类型。

2. **上下文类型推断**  
   如果在调用 `create` 方法时，上下文中的类型信息不够明确，TypeScript 可能会倾向于选择数组版本。

3. **TypeORM 的类型定义问题**  
   `TypeORM` 的类型定义可能不够精确，导致 TypeScript 的类型推断机制无法正确推断。

## 解决方案

### 方法 1：显式类型断言

在调用 `create` 方法时，显式地将参数断言为单个对象，而不是数组。这样可以明确告诉 TypeScript 你传入的是一个对象。

```ts
const user = this.userRepository.create({ ...userData, password: hashedPassword } as User);
```

### 方法 2：避免使用对象展开语法

考虑直接构造实体对象，而不是使用对象展开语法。

```ts
const user = new User();
user.username = userData.username;
user.password = hashedPassword;
user.roles = userData.roles;
```

### 方法 3：检查 `userData` 的类型

确保 `userData` 的类型是明确的，而不是 `any` 或者一个复杂的类型。例如：

```ts
interface UserData {
  username: string;
  password: string;
  roles: string[];
}

async createUser(userData: UserData): Promise<User> {
        const hashedPassword = bcrypt.hashSync(userData.password, 10);
        const user = this.userRepository.create({ ...userData, password: hashedPassword });
        return this.userRepository.save(user);
    }
```


## 总结

三种方法都可以解决,`推荐第三种`
