---
title: NestJS 学习笔记：从0到1实现鉴权的接口
tags: [Node.js, NestJS, REST API, 学习笔记, 踩坑记录]
categories: [后端]
date: 2025-11-15
description: 记录我用 NestJS 从 0 搭出一个简单 REST API 服务的过程，包括项目初始化、模块划分、JWT 登录鉴权，以及中间遇到的一些坑。
articleGPT: 这是一篇 NestJS 学习笔记，围绕“用户登录 + 文章列表”这个简单场景，从搭项目、模块划分、JWT 鉴权到接口调试，记录中间遇到的常见坑和解决思路。
references: []
cover: https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/nestjs.png
---

## 为什么尝试 NestJS？

之前写 Node 后端一般是：

- 直接上 Express / Koa
- 中间件随便堆，越写越乱
- 目录结构靠约定，后期维护成本挺高

看了 NestJS 的官方介绍后发现它几件事很吸引我：

- 有 **模块化**、**依赖注入**、**装饰器** 这些「有点像后端大框架」的东西
- TypeScript 一等公民，类型体验比纯 Express 好很多
- 内置对验证、管道、拦截器、守卫等的支持，做干净的接口更方便

于是就找了一个简单场景来练手：**做一个「用户登录 + 文章列表」的后端服务**。

---

## 场景设计：我想要的后端长什么样？

为了不把自己搞死，第一版的目标控制在：

1. 用户模块 `User`

   - 注册（暂时可以先跳过，手动插数据）
   - 登录：账号 + 密码，成功后返回一个 JWT token

2. 文章模块 `Posts`

   - 获取文章列表（不登录也可以访问）
   - 获取当前登录用户自己的文章（需要登录）
   - 后续可以再加：新增 / 编辑 / 删除（本篇先不展开）

3. 技术点：
   - Nest CLI 初始化项目
   - 使用 `@nestjs/config` 管理环境变量
   - 使用 `class-validator` + `class-transformer` 做 DTO 校验
   - 使用 `@nestjs/jwt` 做登录鉴权（Guard + Strategy）

---

## 第一步：初始化 NestJS 项目（以及我踩的第一个坑）

初始化命令：

```bash
# 全局安装 CLI（可选）
npm i -g @nestjs/cli

# 创建项目
nest new nestjs-blog-api
```

过程中选择包管理器（我选的是 `pnpm`，你也可以用 `npm` 或 `yarn`）。

进入项目目录：

```bash
cd nestjs-blog-api
pnpm install
pnpm run start:dev
```

浏览器打开 `http://localhost:3000`，可以看到默认的 `Hello World!`。

### 踩坑 1：Node / pnpm 版本问题

一开始我用的是比较老的 Node 版本，结果：

- 安装依赖有些包报「不支持当前 Node 版本」
- Nest CLI 也会有警告

**解决：**

- 把 Node 升到 LTS（比如 18+）
- 如果你用 `nvm` 或 `fnm` 管 Node 版本，记得在项目根目录切到正确版本再装依赖

---

## 第二步：拆分模块：Auth + Users + Posts

NestJS 很强调「模块化」，默认项目根里有 `AppModule`。我按场景拆了三个模块：

```bash
nest g module auth
nest g module users
nest g module posts
```

再生成对应的 service / controller：

```bash
nest g service auth
nest g controller auth

nest g service users
nest g controller users

nest g service posts
nest g controller posts
```

此时目录大概是：

```txt
src/
  app.module.ts
  auth/
    auth.module.ts
    auth.service.ts
    auth.controller.ts
  users/
    users.module.ts
    users.service.ts
    users.controller.ts
  posts/
    posts.module.ts
    posts.service.ts
    posts.controller.ts
```

### 踩坑 2：循环依赖

一开始我为了图方便，在 `AuthModule` 里直接引入 `UsersModule`，在 `UsersService` 里又想用 `AuthService` 做点事，结果就出现了 Nest 警告的**循环依赖**。

解决办法其实很简单：**不要一开始就过度纠缠模块之间的调用**。我调整后的设计是：

- `AuthModule` 依赖 `UsersModule`，只在登录时用 `UsersService` 查用户
- `UsersModule` 不再依赖 `AuthModule`

如果真的避免不了循环依赖，可以用 `forwardRef`，但第一版不建议一上来就这么玩。

---

## 第三步：接入环境变量（@nestjs/config）

先安装依赖：

```bash
pnpm add @nestjs/config
```

在 `app.module.ts` 中引入：

```ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { AuthModule } from "./auth/auth.module";
import { UsersModule } from "./users/users.module";
import { PostsModule } from "./posts/posts.module";

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // 全局可用
    }),
    AuthModule,
    UsersModule,
    PostsModule,
  ],
})
export class AppModule {}
```

在项目根目录新建 `.env`：

```env
JWT_SECRET=super_secret_key_change_me
JWT_EXPIRES_IN=7d
```

之后在 `AuthModule` 里就可以通过 `ConfigService` 读这些变量。

---

## 第四步：用户和登录 DTO + 校验（class-validator）

为了让接口出入口更干净，我给登录加了 DTO：

```bash
pnpm add class-validator class-transformer
```

在 `main.ts` 中启用全局验证管道：

```ts
import { ValidationPipe } from "@nestjs/common";
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // 过滤 DTO 里没有声明的字段
      forbidNonWhitelisted: true, // 对多余字段直接报错
      transform: true, // 自动类型转换
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

在 `auth` 模块里新增 `dto/login.dto.ts`：

```ts
import { IsString, MinLength } from "class-validator";

export class LoginDto {
  @IsString()
  username: string;

  @IsString()
  @MinLength(6)
  password: string;
}
```

在 `auth.controller.ts` 中使用：

```ts
import { Body, Controller, Post } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { LoginDto } from "./dto/login.dto";

@Controller("auth")
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post("login")
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }
}
```

### 踩坑 3：忘记开全局验证管道

一开始 DTO 写得很开心，装饰器也加了，但接口始终「什么垃圾参数都能过」。原因就是：

- 只装了 `class-validator`，但没有在 `main.ts` 里用 `ValidationPipe`。

记住：**Nest 只是帮你定义 DTO，真正的校验需要管道来触发**。

---

## 第五步：接入 JWT 鉴权（@nestjs/jwt）

安装依赖：

```bash
pnpm add @nestjs/jwt passport @nestjs/passport passport-jwt
pnpm add -D @types/passport-jwt
```

在 `auth.module.ts` 中配置 JWT：

```ts
import { Module } from "@nestjs/common";
import { JwtModule } from "@nestjs/jwt";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { AuthService } from "./auth.service";
import { AuthController } from "./auth.controller";
import { UsersModule } from "../users/users.module";
import { JwtStrategy } from "./jwt.strategy";

@Module({
  imports: [
    UsersModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.get<string>("JWT_SECRET"),
        signOptions: {
          expiresIn: config.get<string>("JWT_EXPIRES_IN") || "7d",
        },
      }),
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

`auth.service.ts` 中实现 `login`：

```ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { UsersService } from "../users/users.service";
import { LoginDto } from "./dto/login.dto";

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
  ) {}

  async login(dto: LoginDto) {
    const user = await this.usersService.findByUsername(dto.username);
    if (!user) {
      throw new UnauthorizedException("用户不存在");
    }

    // 这里为了简化，直接明文对比密码（真实项目要用 bcrypt）
    if (user.password !== dto.password) {
      throw new UnauthorizedException("密码错误");
    }

    const payload = { sub: user.id, username: user.username };
    const token = await this.jwtService.signAsync(payload);

    return {
      access_token: token,
      user: {
        id: user.id,
        username: user.username,
      },
    };
  }
}
```

定义 `JwtStrategy`（`auth/jwt.strategy.ts`）：

```ts
import { Injectable } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";
import { ConfigService } from "@nestjs/config";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.get<string>("JWT_SECRET"),
    });
  }

  async validate(payload: any) {
    // 这里返回的对象会被挂到 req.user 上
    return { userId: payload.sub, username: payload.username };
  }
}
```

### 踩坑 4：Authorization 头的格式

JWT 守卫默认读取的是：

```txt
Authorization: Bearer <token>
```

一开始我在 Postman 里手懵了，直接写成：

```txt
Authorization: <token>
```

结果一直 401。记得一定要带上 `Bearer ` 前缀，或者自行调整 `jwtFromRequest` 的实现。

---

## 第六步：用守卫保护需要登录的接口

在 `posts` 模块里，假设我有两个接口：

- `GET /posts/public`：公开文章列表
- `GET /posts/mine`：当前登录用户的文章，需要登录

`posts.controller.ts`：

```ts
import { Controller, Get, UseGuards, Request } from "@nestjs/common";
import { JwtAuthGuard } from "../auth/jwt-auth.guard";
import { PostsService } from "./posts.service";

@Controller("posts")
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  @Get("public")
  getPublicPosts() {
    return this.postsService.getPublicPosts();
  }

  @UseGuards(JwtAuthGuard)
  @Get("mine")
  getMyPosts(@Request() req: any) {
    return this.postsService.getPostsByUser(req.user.userId);
  }
}
```

`auth/jwt-auth.guard.ts`：

```ts
import { Injectable } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {}
```

`posts.service.ts` 中简单模拟数据：

```ts
import { Injectable } from "@nestjs/common";

@Injectable()
export class PostsService {
  private posts = [
    { id: 1, title: "公开文章 1", userId: 1, isPublic: true },
    { id: 2, title: "公开文章 2", userId: 2, isPublic: true },
    { id: 3, title: "我的私有草稿", userId: 1, isPublic: false },
  ];

  getPublicPosts() {
    return this.posts.filter((p) => p.isPublic);
  }

  getPostsByUser(userId: number) {
    return this.posts.filter((p) => p.userId === userId);
  }
}
```

现在流程就是：

1. `POST /auth/login` 拿到 token
2. 请求 `GET /posts/mine` 时，在 Header 里带上：
   ```txt
   Authorization: Bearer <token>
   ```
3. 守卫通过后，`req.user` 就会有 `userId`，可以拿来查自己的文章。

---

## 小结：这一轮 NestJS 实战我踩过的坑 & 收获

这一次基于「用户登录 + 文章列表」的小场景，我对 NestJS 有了更实际一点的认识，顺便踩了不少坑：

- **版本问题要先统一**：Node / pnpm / Nest CLI 版本不统一会带来很多莫名其妙的报错；
- **模块划分要尽量单向依赖**：避免一开始就构建循环依赖，`AuthModule` 只依赖 `UsersModule` 即可；
- **DTO + ValidationPipe 是标配**：只有装了验证管道，DTO 上的装饰器才会真的生效；
- **JWT 鉴权要注意 Header 格式**：`Authorization: Bearer <token>` 是默认读取方式；
- **守卫 + Strategy 的配合**：守卫控制「能不能进来」，Strategy 控制「进来后 user 是谁」。

后面我打算在这个基础上继续往下做：

- 接上真实的数据库（比如 MySQL）
- 用 TypeORM / Prisma 管理实体
- 把文章和用户关联起来，做一个简单的「我的后台」

如果你也在学 NestJS，希望这篇笔记能让你少踩几个坑，也欢迎和我一起在「大神之路」上继续折腾。
