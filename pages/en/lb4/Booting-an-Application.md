---
lang: en
title: 'Booting an Application'
keywords: LoopBack 4.0, LoopBack 4
tags:
sidebar: lb4_sidebar
permalink: /doc/en/lb4/Booting-an-Application.html
summary:
---

## What does it mean to boot an Application?

A typical LoopBack application is made up of many artifacts in different files,
organized in different folders. **Booting an Application** means:

* Discover artifacts automatically based on a convention (a specific folder
  containing files with a given suffix)
* Process those artifacts automatically (this usually means binding them to the Application)

`@loopback/boot` provides a Bootstrapper that uses Booters to automatically
discover and bind artifacts, all packaged in an easy to use Mixin.

### What is an artifact?

An artifact is any LoopBack construct usually defined in code as a Class. The word
artifact describes Controllers, Repositories, Models, etc.

## Usage: @loopback/cli

New projects generated using `@loopback/cli` or `lb4` are automatically enabled
to use `@loopback/boot` for booting the Application using the conventions
followed by the CLI.

---

The rest of this page describes the inner workings of `@loopback/boot` for advanced use
cases, manual usage or using `@loopback/boot` as a standalone package (with custom
booters).

## BootMixin

Boot functionality can be added to a LoopBack 4 Application by mixing it with the
`BootMixin`. The Mixin adds the `BootComponent` to your Application as well as
convenience methods such as `app.boot()` and `app.booters()`. The Mixin also allows
Components to set the property `booters` as an Array of `Booters`. They will be bound
to the Application and called by the `Bootstrapper`.

Since this is a convention based Bootstrapper, it is important to set a `projectRoot`,
as all other artifact paths will be resolved relative to this path.

_Tip_: `application.ts` will likely be at the root of your project, so it's path can be
used to set the `projectRoot` by using the `__dirname` variable. _(See example below)_

#### Using the BootMixin

`Booter` and `Binding` types must be imported alongside `BootMixin` to allow TypeScript
to infer types and avoid errors. \_If using `tslint` with the `no-unused-variable` rule,
you can disable it for the import line by adding `// tslint:disable-next-line:no-unused-variable`.

```ts
import { BootMixin, Booter, Binding } from "@loopback/boot";

class MyApplication extends BootMixin(Application) {
  constructor(options?: ApplicationConfig) {
    super(options);
    // Setting the projectRoot
    this.projectRoot = __dirname;
  }
}
```

### app.boot()

A convenience method to retrieve the `Bootstrapper` instance bound to the
Application and calls it's `boot` function. This should be called before an
Application's `start()` method is called. _This is an `async` function and should
be called with `await`._

```ts
class MyApp extends BootMixin(Application) {}

async main() {
  const app = new MyApp();
  app.projectRoot = __dirname;
  await app.boot();
  await app.start();
}
```

### app.booters()

A convenience method manually bind `Booters`. You can pass any number of `Booter`
classes to this method and they will all get bound to the Application using the
prefix and tag used by the `Bootstrapper`.

```ts
// Binds MyCustomBooter to `booters.MyCustomBooter`
// Binds AnotherCustomBooter to `booters.AnotherCustomBooter`
// Both will have the `booter` tag set.
app.booters(MyCustomBooter, AnotherCustomBooter);
```

## BootComponent

This component is added to a Application by `BootMixin` if used. The Component:

* Provides a list of default `booters` as a property of the component
* Binds the conventional Bootstrapper to the Application

_If using this as a standalone component without the `BootMixin`, you will need to
bind the `booters` of a component manually._

```ts
app.component(BootComponent);
```

## Bootstrapper

A Class that acts as the "manager" for Booters. The Boostrapper is designed to be
bound to an Application as a `SINGLETON`. The Bootstrapper class provides a `boot()`
method. This method is responsible for getting all bound `Booters` and running
their `phases`. A `phase` is a method on a `Booter` class.

Each call of the `boot()` method creates a new `Context` that sets the `app` context
as it's parent. This is done so each `Context` for `boot` gets a new instance of
`booters` but the same context can be passed into `boot` so selective `phases` can be
run in different calls of `boot`.

The boostrapper can be configured to run only certain booters or phases of booters
by passing in `BootExecOptions`. **This is experimental and subject to change. Hence,
this functionality is not exposed when calling `boot()` via `BootMixin`**.

To use `BootExecOptions` you must directly call `bootstrapper.boot()` instead of `app.boot()`.
You can pass in the `BootExecOptions` object with the following properties:

| Property         | Type                    | Description                                      |
| ---------------- | ----------------------- | ------------------------------------------------ |
| `booters`        | `Constructor<Booter>[]` | Array of Booters to bind before running `boot()` |
| `filter.booters` | `string[]`              | Names of Booter classes that should be run       |
| `filter.phases`  | `string[]`              | Names of Booter phases to run                    |

### Example

```ts
class MyApp extends BootMixin(Application) {}
const app = new MyApp();
app.projectRoot = __dirname;

const bootstrapper = await this.get(BootBindings.BOOTSTRAPPER_KEY);
bootstrapper.boot({
  booters: [MyCustomBooter],
  filter: {
    booters: ["MyCustomBooter"],
    phases: ["configure", "discover"] // Skip the `load` phase.
  }
});
```

## Booters

A Booter is a Class that implements the `Booter` interface. The Class
must implement methods that corresponds to a `phase` name. The `phases` are called
by the Bootstrapper in a pre-determined order (unless overridden by `BootExecOptions`).
The next phase is only called once the previous phase has been completed for all Booters.

### Phases

#### configure

Used to configure the `Booter` with it's default options.

#### discover

Used to discover the artifacts supported by the `Booter` based on convention.

#### load

Used to bind the discovered artifacts to the Application.
