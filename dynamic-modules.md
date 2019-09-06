### Dynamic modules

The [Modules chapter](/modules) covers the basics of Nest modules, and includes a brief introduction to **Dynamic modules**. This chapter expands on the subject of Dynamic modules. Upon completion, you should have a good grasp of what they are and how and when to use them.

#### Introduction

Most user code examples in the **Overview** section of the documentation make use of regular, or _static_ modules. Modules define groups of components like providers and controllers that fit together as a modular part of an overall application. They provide an execution context, or _scope_, for these components. For example, providers defined in a module are visible to other members of the module without the need to export them. When a provider needs to be visible outside of a module, it is first _exported_ from its host module, and then _imported_ into its consuming module.

Let's walk through a familiar example.

First, we'll define a `UsersModule` to provide and export a `UsersService`. `UsersModule` is the _host_ module for `UsersService`.

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Next, we'll define an `AuthModule` to consume the exported `UsersService`.

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

These constructs allow us to inject `UsersService` in, for example, the `AuthService` that is hosted in `AuthModule`:

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private readonly usersService: UsersService) {}
  /*
    Implementation that makes use of this.usersService
  */
}
```

We'll refer to this as _static_ module binding. All the information Nest needs to wire together the modules has already been declared in the host and consuming modules. Let's review this in a bit more detail. Nest makes `UsersService` available inside `AuthModule` by:

1. Instantiating `UsersModule`, including transitively importing other modules that `UsersModule` itself consumes, and transitively resolving any dependencies ([see Custom providers](https://docs.nestjs.com/fundamentals/custom-providers)).
2. Instantiating `AuthModule`, and making `UsersModule`'s exported providers available to components in `AuthModule` (just as if they had been declared in `AuthModule`).
3. Injecting an instance of `UsersService` in `AuthService`.

#### Dynamic module use case

With static module binding, there's no opportunity for the consuming module to **influence how providers from the host module are configured**. Why does this matter? Consider the case where we have a general purpose module that needs to behave differently in different use cases. This is analogous to the concept of a "plugin" in many systems, where a generic facility requires some configuration to be used by a consumer.

A good example with Nest is a **configuration module**. Many applications find it useful to externalize configuration details by using a configuration module. This makes it easy to dynamically change the configuration with different deployments: e.g., a development database for developers, a staging database for the staging/testing environment, etc. By delegating the management of configuration parameters to a configuration module, the application source code remains independent of configuration parameters.

The challenge is that the configuration module itself, since it's generic, needs to be _configured_ by its consuming module. This is where _Dynamic modules_ come into play. Using dynamic module features, we can make our configuration module **dynamic** so that the consuming module has an opportunity to influence how it is configured.

In other words, dynamic modules provide an API for importing one module into another, as opposed to using the static bindings we've seen so far.

#### Configuration module example

We'll be using the basic version of the [configuration example](/techniques/configuration#service) for this section. The completed version is available as a working [example here]().

Our requirement is to make `ConfigModule` configurable. The basic sample hard-codes the location of the `.env` file to be in the project root folder. Let's suppose we want to make that configurable, such that you can manage your `.env` files in another folder. For example, suppose you want to store your various `.env` files in a folder under the project root called `config` (a sibling folder to `src`).

Dynamic modules give us the ability to pass configuration parameters into the module being imported. Let's start from the end-goal of how this might look, and then work backwards. First, let's quickly review the _static_ case (i.e., one which has no ability to configure the imported module):

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

A _dynamic module_ import, where we're passing in a configuration object, might look like this:

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [
    ConfigModule.register({
      folder: `${path.join(__dirname, '../../config')}`,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Let's take a look at what's happening here. What are the moving parts?

1. `ConfigModule` is a normal class, so we can see that it must have a **static method** called `register()`. We know it's static because we're calling it on the **class**, not on an **instance** of the class. This method, which we will create, can have any arbitrary name, but conventionally it's called something like `forRoot()` or `register()`.
2. The `register()` method must return something _like_ a `module` since it's return value appears in the familiar `imports` list, which we've seen so far includes a list of modules.

In fact, what our `register()` method will return is a `DynamicModule`. This construction is the result of the dynamic module API we were referring to earlier. Let's take a look at the `DynamicModule` interface.

```typescript
interface DynamicModule extends ModuleMetadata {
  module: Type<any>;
}
```

Just so we can see the full interface, let's imagine we could collapse it to also include the inherited properties from `ModuleMetadata`, which it extends. Our full interface for a `DynamicModule` would then look like:

```typescript
interface DynamicModule {
  module: Type<any>;
  imports?: Array<
    Type<any> | DynamicModule | Promise<DynamicModule> | ForwardReference
  >;
  controllers?: Type<any>[];
  providers?: Provider[];
  exports?: Array<
    | DynamicModule
    | Promise<DynamicModule>
    | string
    | symbol
    | Provider
    | ForwardReference
    | Abstract<any>
    | Function
  >;
}
```

Looking at this, we can now start to easily map the interface of a `DynamicModule` to the metadata properties associated with a normal (what we've been calling _static_) module. Aside from the additional `module` property, which we'll deal with shortly, the other properties map 1 to 1 with the properties in our familiar `@Module()` decorator. From this we can see that dynamic modules share the exact same properties (plus the additional `module` property) as statically declared modules. This is a key observation.

What about the static `register()` method? We can now see that its job is to return an object that has the `DynamicModule` interface. By so doing, we are effectively providing a module to the `imports` list, similar to the same way we would by listing a module class name. This isn't 100% accurate, but the concept should make sense. There are still two differences to understand to make the picture complete:

1. We can now see that the `@Module()` decorator's `imports` property can take not only a class name (e.g., `imports: [UsersModule]`), but also a function **returning** a dynamic module (e.g., `imports: [ConfigModule.register({...})]`).
2. A dynamic module has an additional property, called `module`, which serves as its name. More on this in a moment.

Now we can begin to infer what our dynamic `ConfigModule` must look like. Let's take a crack at it.

```typescript
// src/config/config.module.ts
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: ConfigService,
      exports: ConfigService,
    };
  }
}
```

It should now be clear how the pieces tie together. Calling `ConfigModule.register()` returns a `DynamicModule` object with properties which are essentially the same as those that, thus far, we've provided as metadata via the `@Module()` decorator. In other words, it returns a module.

> Hint: be sure to import `DynamicModule` from `@nestjs/common`

Our dynamic module isn't terribly interesting yet, as we haven't introduced any capability to **configure** it as we said we would like to do. Let's tackle that next.

### Module configuration

The obvious solution for customizing the behavior of the `ConfigModule` is to pass it an options object in the static `register()` method. Looking again from the perspective of the consuming module, our interface should look like this:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [
    ConfigModule.register({
      folder: `${path.join(__dirname, '../config')}`,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

So now we know how to pass an options object to our dynamic module. How do we use it? Let's consider that for a minute. We know that our `ConfigModule` is basically a host for providing an injectable service - the `ConfigService` - for use by other providers. Let's take a moment to review the `ConfigService` below:

```typescript
// src/config/config.service.ts
import * as dotenv from 'dotenv';
import * as fs from 'fs';

import { EnvConfig } from './interfaces';

export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const filePath = `${
      process.env.NODE_ENV ? process.env.NODE_ENV : 'development'
    }.env`;
    this.envConfig = dotenv.parse(fs.readFileSync(filePath));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Let's assume for the moment that we know how to get the options object that was passed in via `register()` into our `ConfigService`. Let's make a few changes to our `ConfigService` to use that object. Since we _haven't_ actually passed it in, we'll just hard code it for now.

```typescript
// src/config/config.service.ts
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';

import { EnvConfig } from './interfaces';

export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = {
      folder: '../config',
    };

    const filePath = `${
      process.env.NODE_ENV ? process.env.NODE_ENV : 'development'
    }.env`;

    const envFile = path.resolve(__dirname, '../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Our only remaining task is to somehow _inject_ the options object into our `ConfigService`. And of course, _injection_ is the way we'll do it. Remember from the **Custom providers** chapter that [providers can include any value, not just services](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers). What we'll do is bind our options object to the Nest IoC container, and inject it into `ConfigService`. Let's tackle the first part.

Returning to our `register()` method, we now know that the properties it returns are the same as the metadata for any module, including any _providers_ that are needed by the module. So, we want to bind our options object as a provider to an injection token. This will make it injectable into the service -- which we'll take advantage of in the next step.

```typescript
// src/config/config.module.ts
import { DynamicModule, Module } from '@nestjs/common';

import { ConfigService } from './config.service';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {
  static register(options): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
      ],
    };
  }
}
```

Now we can complete the process by injecting `'CONFIG_OPTIONS'` into the `ConfigService` constructor:

```typescript
import { Inject } from '@nestjs/common';

import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';

import { EnvConfig, ConfigOptions } from './interfaces';

export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: ConfigOptions) {
    const filePath = `${
      process.env.NODE_ENV ? process.env.NODE_ENV : 'development'
    }.env`;

    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

One final note, for simplicity we used a string-based injection token (`'CONFIG_OPTIONS'`) above, but best practice is to define it as a constant in a separate file, and import that file, as such:

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```
