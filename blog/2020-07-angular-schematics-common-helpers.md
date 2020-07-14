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

## Create an Angular Schematics example project

First things first: We need some project where we try out things.
You can either use an existing schematics project or simply create a new blank one:

```bash
npx @angular-devkit/schematics-cli blank --name=playground
```

> If you are not familar with the basics of authoring schematics, I would recommend you to read the [Angular Docs](https://angular.io/guide/schematics-authoring) and the [blog post _"Total Guide To Custom Angular Schematics"_ from Tomas Trajan](https://medium.com/@tomastrajan/total-guide-to-custom-angular-schematics-5c50cf90cdb4) first.

After setting up the new blank project we should have this file available: `src/playground/index.ts`.
 
```ts
import { Rule, SchematicContext, Tree } from '@angular-devkit/schematics';

export function playground(_options: any): Rule {
  return (tree: Tree, _context: SchematicContext) => {
    return tree;
  };
}
```

This will be our base for the following examples and explanations.

### Basic types

In case you are not familar with the schematics, I will just explain some very basic things in short:

| type | explanation |
|---|---|
|`Tree`|Is the structured virtual representation of every file in the workspace|
|`Rule`|Is called with a `Tree` and a `SchematicContext`. The Rule is supposed to make changes on the `Tree` and returns the adjusted `Tree`|
|`SchematicContext`|Contains information necessary for Schematics to execute some rules|

### `@schematics/angular`
A second thing we need to do is to install the package `@schematics/angular` which contains all the utils we need for the further steps.
This package contains all the schematics the Angular CLI uses by itself when running command like `ng generate` or `ng new` etc.

```bash
npm i --save @schematics/angular
```

## Get, Add and Remove (dev-, peer-) dependencies of/in the `package.json` file

A very common thing when authoring a schematic is adding a dependency to the `package.json` file.
Of course, we can implement a function that parses and writes to/from our JSON file.
But why should we solve a problem that's already solved?

We can simply use the functions from `@schematics/angular/utility/dependencies` to handle such dependency operations.
The function `addPackageJsonDependency()` allows us to add a dependency object of type `NodeDependency` to the `package.json` file.
The property `type` must contain a value of the `NodeDependencyType` enum.
Behind it's values we will find the different sections names a `package.json` file can contain.`'dependencies'`, `'devDependencies'`, `'peerDependencies'` and `'optionalDependencies'`.
As first parameter it needs the `Tree` with all it's files.

We can use the `getPackageJsonDependency()` function to request the dependency configuration as an `NodeDependency` object.
The good thing here: We don't need to know if we will request d dependency form the `dependencies`, `devDependencies`, `peerDependencies` or `optionalDependencies` section of the `package.json` file.

The third function I want to show you is `removePackageJsonDependency()`.
As `getPackageJsonDependency()` it needs also just the `Tree` and the package name and it will remove this dependency from the correct section in the `package.json` file.

By default all this functions will use the `package.json` file in the root of the tree, but we can pass a third parameter containing a specific path to the `package.json` file.

```ts
import { Rule, SchematicContext, Tree } from '@angular-devkit/schematics/';
import {
  NodeDependency,
  NodeDependencyType
  getPackageJsonDependency,
  addPackageJsonDependency,
  removePackageJsonDependency,
} from '@schematics/angular/utility/dependencies';

export function playground(_options: any): Rule {
  return (tree: Tree, _context: SchematicContext) => {
    const dep: NodeDependency = {
      type: NodeDependencyType.Dev,
      name: 'my-package',
      version: '~0.3.4-beta.1',
      overwrite: true,
    };

    addPackageJsonDependency(tree, dep);
    console.log(getPackageJsonDependency(tree, 'my-package'))
    // { type: 'dependencies', name: 'typescript', version: '~3.9.2' }

    removePackageJsonDependency(tree, 'my-package');
    console.log(getPackageJsonDependency(tree, 'my-package'))
    // null

    return tree;
  };
}
```

> [Checkout the implementations in detail](https://github.com/angular/angular-cli/blob/master/packages/schematics/angular/utility/dependencies.ts)

## Add a content to a specific position

Sometimes we need to change some contents of a file.
Indenpendently from the file type of the file, we can use the `InsertChange` class that'll return a change object containing the content to be added and the position where the changes is inserted.
In the following example we will create a new file called `my-file.fileextension` with the content '`const a = 'foo';`' inside the virtual tree.
Now we will instanciate a new `InsertChange` with the filepath, the position where we want to add the change and finally the content we want to add.
The next step for us is to start the update process for the file using the `beginUpdate` method on our tree.
This method returns an `UpdateRecorder`.
We use the `insertLeft()` method and we will hand over the postion and the content (`toAdd`) from the `InsertChange`.
The changes is now marked but not proceeded.
To really update the files content, we need to call the `commitUpdate()` method on our tree with the `exportRecorder`.
When we will now call `tree.get(filePath)`, we can log the files content and see that the change has been proceeded.
To delete a file inside the virtual tree, we can use the `delete()` method with the filepath on the tree.

```ts
import { Rule, SchematicContext, Tree } from '@angular-devkit/schematics/';
import { InsertChange } from '@schematics/angular/utility/change';

export function playground(_options: any): Rule {
  return (tree: Tree, _context: SchematicContext) => {
    const filePath = 'my-file.fileextension';
    tree.create(filePath, `const a = 'foo';`);

    // insert a new change
    const insertChange = new InsertChange(filePath, 16, '\nconst b = \'bar\';');
    const exportRecorder = tree.beginUpdate(filePath);
    exportRecorder.insertLeft(insertChange.pos, insertChange.toAdd);
    tree.commitUpdate(exportRecorder);
    console.log(tree.get(filePath)?.content.toString())
    // const a = 'foo';
    // const b = 'bar';

    tree.delete(filePath); // cleanup (if not running schematic in debug mode)
    return tree;
  };
}
```

## Add or Remove TypeScript imports



