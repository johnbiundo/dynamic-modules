### Dynamic modules

The [Modules chapter](/modules) covers the basics of Nest modules, and includes a brief introduction to **Dynamic modules**.  This chapter expands on the subject of Dynamic modules.  Upon completion, you should have a good grasp of what they are and how and when to use them.

#### Introduction

Most user code examples in the **OVERVIEW** section of the documentation make use of regular, or *static* modules.  These modules, as we've seen, define groups of components like providers and controllers that fit together as a modular part of an overall application.  As we've seen, modules provide an execution context, or *scope*, for these components.  For example, providers defined in a module are visible to other members of the module without the need to export them.  When a provider needs to be visible outside of a module, it is first *exported* from its host module, and then *imported* into its consuming module.

Let's walk through a familiar example.

First, we'll define a `UsersModule` to provide and export a `UsersService`.  `UsersModule` is the *host* module for `UsersService`.

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

We'll refer to this as *static* module binding.  All the information Nest needs to wire together the modules has already been declared in the host and consuming modules.  Let's walk through this in a bit more detail. Nest makes `UsersService` available inside `AuthModule` by:
1. Instantiating `UsersModule`, including transitively importing other modules that `UsesModule` itself consumes, and transitively resolving any dependencies ([see Custom providers](https://docs.nestjs.com/fundamentals/custom-providers)).
2. Instantiating `AuthModule`, and making `UsersModule`'s exported providers available inside `AuthModule` (just as if they had been declared in `AuthModule`).

#### Dynamic module use case
With static module binding, there's no opportunity for the consuming module to **influence how providers from the host module are configured**.  Why does this matter?  Consider the case where we have a general purpose module that needs to behave differently in different use cases.  This is analogous to the concept of a "plugin" in many systems, where a generic facility requires some configuration to be used by a consumer.

A good example with Nest is a **configuration module**.  Many applications find it useful to externalize configuration details by using a configuration module.  This makes it easy to dynamically change the configuration with different deployments: e.g., a development database for developers, a staging database for the staging/testing environment, and a production database for the production environment.  By delegating the management of configuration parameters to a configuration module, the application source code remains independent of configuration parameters.

The challenge is that the configuration module itself, since it's generic, needs to be *configured* by its consuming module.  This is where *Dynamic modules* come into play.  We can make our configuration module **dynamic** so that the consuming module has an opportunity to influence how it is configured.

In other words, Dynamic modules provide an API for importing one module into another, as opposed to using the static bindings we've seen so far.

#### Configuration module example

We'll be using the [configuration sample found here](https://github.com/nestjs/nest/tree/master/sample/25-configuration) for this section.

Our requirement is to make `ConfigModule` configurable.  Its current version hard codes the location of the `.env` file to be in the project root folder.  Let's suppose we want to make that configurable, so that you can manage your `.env` files in a folder that's in a path *relative* to the root folder.  For example, suppose you want to store your various `.env` files in a folder under the project root called `config` (a sibling folder to `src`).

Dynamic modules give us this capability by allowing us to pass in configuration parameters to the module being imported. Let's start with how this might look.  First, let's quickly review the *static* case (which has no ability to configure the import):

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { ConfigModule } from '../config/config.module';
@Module({
  imports: [UsersModule, ConfigModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

A *dynamic module* import, where we're passing in a configuration object, might look like this:

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { ConfigModule } from '../config/config.module';
@Module({
  imports: [
    UsersModule,
    ConfigModule.register({
      folder: `${path.join(__dirname, '../config')}`,
    })
  ],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

Let's walk through what is happening here. What are the moving parts?
1. `ConfigModule` is a normal module class, so it apparently must have a **static method** called `register()`.  This method, which you create, can have any arbitrary name, but conventionally it's called something like `forRoot()` or `register()`.  We know it's static because we're calling it on the **class**, not on an **instance** of the class.
2. The `register()` module must return something *like* a `module` since it appears in the familiar `imports` list, which we've seen so far includes a list of modules.

In fact, what our `register()` method will return is a `DynamicModule`.  This is the API we were referring to earlier.  Let's take a look at the `DynamicModule` interface.

```typescript
interface DynamicModule extends ModuleMetadata {
  module: Type<any>;
}
```

To see the full interface, let's temporarily imagine we can collapse the interface to see the inherited properties from `ModuleMetadata` (we do this for explanatory purposes; in fact, the elements of `DynamicModule` are declared separately as above, and combined with those it inherits from the `ModuleMetadata` interface using normal TypeScript inheritance).  Our full interface for a `DynamicModule` thus looks like:

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

Interestingly, we can now start to easily map the interface of a `DynamicModule` to the metadata properties associated with a normal module.  Aside from the `module` property, which we'll deal with shortly, the others map 1 to 1 with the properties in our familiar `@Module()` decorator.  From this we can see that dynamic modules share the exact same properties (plus the additional `module` property) as statically declared modules.  This is a key observation.

What about that `register()` static method?  We can now see that its job is to return an object that has the `DynamicModule` interface.  By so doing, we are effectively providing a module to the `imports` list in the same way we would by listing a module class name.  This isn't 100% accurate, but the concept should make sense.  There are two differences to understand to make the picture complete:

1. We can now see that the `@Module()` decorator's `imports` property can take not only a class name (e.g., `imports: [UsersModule]`), but also a function **returning** a dynamic module (e.g., `imports: [ConfigModule.forRoote({...})]`).
2. A dynamic module has an additional property, called `module`, which serves as a name for the dynamically created module.  More on this in a moment.

Now we can begin to infer what our dynamic `ConfigModule` must look like.  Let's take a crack at it.

```typescript
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
    }
  }
}
```

It should now be clear how the pieces tie together.  Calling `ConfigModule.register()` returns a `DynamicModule` object with properties which are essentially the same as those that, thus far, we've provided as metadata via the `@Module()` decorator.  In other words, it returns a module.

Our dynamic module isn't terribly interesting yet, as we haven't introduced any capability to **configure** it as we said we would like to do.  Let's tackle that next.



