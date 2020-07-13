---
title: 'Author Angular Schematics by using common helpers'
description: 'Authoring an Angular CLI Schematic offers us a way to add, scaffold and update app releated modules and files. In this article I will guide you through some common but currently undocumented helper functions you can use to acheive your goal'
published: false
author: Danny Koppenhagen
mail: mail@d-koppenhagen.de
updated: 2020-07-13
keywords:
  - Angular
  - Angular CLI
  - Schematics
language: en
thumbnail: assets/images/blog/schematics-helpers/schematics-helpers.jpg
thumbnailSmall: assets/images/blog/schematics-helpers/schematics-helpers-small.jpg
---

Authoring an Angular CLI Schematic offers us a way to add, scaffold and update app releated modules and files. However there are some common things such as updating your `package.json` file, adding or removing an Angular module or updating component imports we will probably integrate in our schematics.

Currently the way of authoring an Angular Schematic is documented at [angular.io](https://angular.io/guide/schematics-authoring).
But there is one big thing missing in the documentation: The way of integrating typical tasks.
The Angular CLI itself uses the schematics for e.g. generating modules and components, adding imports or modifying the `package.json`.
Under the hood each of the schematics uses some very common utils that are not documentd by now but still available for all developers.
As I've see some Angular CLI schematic projects where people trying to implement almost the same common utils methods by theirselfs as already implemented in the Angular CLI, I want to show you in this blog post some very typical helpers that you can use for you Angular CLI Schematic porject to prevent any pitfalls.

## Get, Add and Remove (dev-, peer-) dependencies of/in the `package.json` file

A very common thing when authoring a schematic is adding a dependency to the `package.json` file.
Of course we can implement a function that parses and writes to/from our JSON file.
But why should we solve a problem that's already solved?

> [Checkout the implementations in detail](https://github.com/angular/angular-cli/blob/master/packages/schematics/angular/utility/dependencies.ts)

### Get a dependency

The first method you'll probably use is the `getPackageJsonDependency`.
This will return an object with the requested dependency configuration.
It tells you the type of the dependency which can be either `dependencies`, `devDependencies`, `peerDependencies` or `optionalDependencies`.
The function has also a third parameter where you can specify the path to the `package.json` file optionally.
By default it uses the `package.json` file in the project root.

```ts
import { getPackageJsonDependency } from '@angular-devkit/schematics';

const dependency = getPackageJsonDependency(tree, '@angular/core', /* optional: */ 'src/foo/package.json');
console.log(dependency)
// { type: 'dependencies', name: '@angular/core', version: '^10.0.0' }
```

### Add a dependency

In almost the same way, we can also add a new dependency to the `package.json` file.
We must simply hand over the dependency object.
The `type` property can also be either `dependencies`, `devDependencies`, `peerDependencies` or `optionalDependencies`.
Also we can spcify the last parameter with the path of the `package.json` file.

```ts
import { addPackageJsonDependency } from '@angular-devkit/schematics';

const dependency = {
  type: 'devDependencies',
  name: 'my-package',
  version: '~0.3.4-beta.1',
  overwrite: true,
}

addPackageJsonDependency(tree, dependency, /* optional: */ 'src/foo/package.json');
```

### Remove a dependency

The last very common function to handle dependencies in the `package.json` file is the `removePackageJsonDependency` function.
This will simply remove the dependency doesn't matter if it's typeof `dependencies`, `devDependencies`, `peerDependencies` or `optionalDependencies`.

```ts
import { getPackageJsonDependency } from '@angular-devkit/schematics';

removePackageJsonDependency(tree, '@angular/core', /* optional: */ 'src/foo/package.json');
```

## Add or Remove TypeScript imports

