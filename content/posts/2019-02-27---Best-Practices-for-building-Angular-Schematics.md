---
title: Best Practices for building Angular Schematics
date: '2019-02-27T22:40:32.169Z'
template: 'post'
draft: false
slug: '/posts/angular-schematics-best-practices/'
category: 'Angular Schematics'
tags:
  - 'Angular'
  - 'Libraries'
  - 'Web Development'
  - 'Angular CLI'
  - 'Angular Schematics'
  - 'TypeScript'
description: 'Thoughts on best practices to use when building custom Angular Schematics'
---

**Note: This article was originally published on the [ng-conf Medium publication.](https://medium.com/ngconf/best-practices-for-building-angular-schematics-419b54057383)**

Angular Schematcs are one of the coolest things to come out of Angular in the past while. With a little bit of work you can build extensions for the Angular CLI that can install and setup libraries in your project, generate code, or make configuration changes easily and in a standard way. They can save you a lot of time in addition to help enforcing coding standards your organization may have.

Here are some best practices that can help save you some time and headaches as you build some awesome schematics.

## Use a sandbox project to test your schematics' functionality as you develop

Testing the functionality of your schematics to make sure they behave the way you expect can be tricky. It usually involves having to link your project to another Angular application and then going back and resetting things after you run your schematics before you try again. This can be cumbersome and much more time than I would like. What if I told you it didn't have to be that way?

[Kevin Schuchard has a great post](https://www.kevinschuchard.com/blog/2018-11-20-schematic-sandbox/) where he reccomends packaging your schematics with its own Angular app, which he calls a sandbox, and wire it up with scripts for quick testing and resetting of the project between runs. This helps you move quickly but, even better, helps others to be productive when working on your schematics later on. I highly suggest you go read the article, but here's a quick and dirty setup of a sandbox for your schematics.

1. Generate an Angular workspace with `ng new sandbox --skipGit`.
2. Add these two scripts to the sandbox's `package.json`

```
"build": "ng build --prod --progress=false",
"test": "ng test --watch=false"
```

3. Update the scripts in the **root** `package.json` with the following. For this example we'll say that we want to test our ng-add schematic.

```
"build": "tsc -p tsconfig.json",
"test": "npm run sandbox:ng-add && npm run test:sandbox",
"clean": "git checkout HEAD -- sandbox && git clean -f -d sandbox",
"link:schematic": "npm link && cd sandbox && npm link my-awesome-schematic",
"sandbox:ng-add": "cd sandbox && ng g my-awesome-schematic:ng-add",
"test:sandbox": "cd sandbox && npm run lint && npm run build"
```

4. Go ahead and setup/commit everything to your git repository at the root of the project (there shouldn't be a secondary one in the sandbox). These scripts will allow us to link our schematics via `npm link` to the sandbox and execute them easily with `npm test` then clean up the sandbox back to it's original state with `npm run clean`.

## Type your schemas

Schemas are great. They allow you to define what options your schematic is expecting and use dialogs to gather information from the user instead of forcing the user to remember various flags for your schematic. After I create a `schema.json` file for my schematics I find it helpful to create an interface that represents that schematic's options.

For example, given this schematic's `schema.json` file:

```json
{
  "$schema": "http://json-schema.org/schema",
  "id": "my-awesome-schematic",
  "title": "My awesome schematic",
  "type": "object",
  "properties": {
    "project": {
      "type": "string",
      "description": "The name of the project.",
      "$default": {
        "$source": "projectName"
      }
    },
    "path": {
      "type": "string",
      "default": "./files",
      "description": "The location you want your files created",
      "x-prompt": "Where would you like your files setup?"
    },
    "moduleName": {
      "type": "string",
      "description": "The module the changes should be made to",
      "x-prompt": "Which module would you like updated?"
    }
  },
  "required": ["moduleName"],
  "additionalProperties": false
}
```

You would create a `schema.model.ts` file like so:

```ts
export interface Schema {
  project: string;
  path: string;
  moduleName: string;
}
```

This way you can use this as the type of the options you pass into the rule factory and you get all the great type checking benefits of Typescriot when working with your rule factorites.

```typescript
import { Rule, SchematicContext, Tree } from '@angular-devkit/schematics';

import { Schema } from './schema.model.ts';

export function myAwesomeSchematic(_options: Schema): Rule {
  return (tree: Tree, _context: SchematicContext) => {
    return tree;
  };
}
```

## Organize your project by schematic

This may seem like a no-brainer, even the example project gernerated when you run `schematics schematic` organizes files like this, but it really is the best way to organize your schematics and adhereing to this will save you a lot of headache down the road. For example if my project had three different schematics (let's call them schematic-a, schematic-b, and schematic-c) you should organize your project like so:

```
src
├── collection.json
├── schematic-a
│   ├── files
│   │   ├── file-1.txt
│   │   └── file-2.txt
│   ├── index.ts
│   ├── index_spec.ts
│   ├── schema.json
│   └── schema.model.ts
├── schematic-b
│   ├── index.ts
│   ├── index_spec.ts
│   ├── schema.json
│   └── schema.model.ts
└── schematic-c
    ├── index.ts
    ├── index_spec.ts
    ├── schema.json
    └── schema.model.ts
```

This is similar to how the Angular style guide reccomends organizing your projects into modules by feature.

## Create helper schematics

The great thing about schematics is that they can call other schematics! This means you can split up your funcitonality among multiple schematics and make them more modular. For example if I had a schematic that installed some libraries then
modified something in the file tree I would create one schematic to handle the installation of the dependencies, one to deal with the file system interactions, and a third to serve as the main schematic that chains the previous two.

Here is what the primary schematic would look like:

```typescript
import {
  Rule,
  SchematicContext,
  Tree,
  chain
} from '@angular-devkit/schematics';

import { Schema } from './schema.model.ts';

export function myAwesomeSchematic(_options: Schema): Rule {
  return (tree: Tree, _context: SchematicContext) => {
    return chain([
      addDependencies(options),
      addRequiredFiles(options)
    ])(tree, _context);
  };
}

function addDependencies(options: Schema): Rule {
  return (tree: Tree, _context: SchematicContext) => {
    _context.addTask(new RunSchematicTask('add-dependencies', options));
    return tree;
  };

function addRequiredFiles(options: Schema): Rule {
  return (tree: Tree, _context: SchematicContext) => {
    _context.addTask(new RunSchematicTask('add-files', options));
    return tree;
  };
```

Then in `collection.json` you can mark those other two schematics as private so they aren't accessable outside of the schematic:

```json
{
  "$schema": "../node_modules/@angular-devkit/schematics/collection-schema.json",
  "schematics": {
    "bootstrap-project": {
      "description": "Main entry point for the schematic. Calls additional schematics and coordinates their operation",
      "factory": "./bootstrap-project/index",
      "schema": "./bootstrap-project/schema.json"
    },
    "add-dependencies": {
      "description": "Install dependencies for the setup process",
      "private": true,
      "factory": "./add-dependencies/index",
      "schema": "./add-dependencies/schema.json"
    },
    "add-files": {
      "description": "Performs additional setup actions for configuring the project.",
      "private": true,
      "factory": "./add-files/index",
      "schema": "./add-files/schema.json"
    }
  }
}
```

## Use the `files` property of `package.json` to only publish what you want

As your project grows and becomes more complex you might have more files and directories added to it. This can be great and ease our lives as developers but a lot of that other stuff we use for developing our schematics should not be packaged with the schematics when we publish to npm for others to use (our fancy new sandbox for example). Those files will never be used by end users and can bloat the footprint of our schematics package.

Luckily theres a way to keep our publish process small and simple without needing to change a lot to the project. Simply add the directories you want published to a `files` array in your `package.json` and let npm take care of the rest! For example, if I only wanted to publish my `src` directory when I released my schematics package I would first create a `.npmignore` that ensures that none of your \*.ts except for definition files will be packaged:

```
# Ignores TypeScript files, but keeps definitions.
*.ts
!*.d.ts
```

Then update the **root** `package.json` like so:

```json
{
  "name": "my-schematic",
  "version": "0.0.0",
  "description": "A blank schematics",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "test": "npm run build && jasmine src/**/*_spec.js"
  },
  "keywords": ["schematics"],
  "files": ["src"], <-- this is the field you add
  "license": "MIT",
  "schematics": "./src/collection.json",
  "dependencies": {
    "@angular-devkit/core": "^7.3.1",
    "@angular-devkit/schematics": "^7.3.1",
    "@types/jasmine": "^3.0.0",
    "@types/node": "^8.0.31",
    "jasmine": "^3.0.0",
    "typescript": "~3.2.2"
  }
}
```

Adding this additional field will tell npm to only package those files that are not ignored within the `src` directory. Hooray smaller package sizes!

## Conclusion

Schematics are very exciting and very powerful. There are a lot of techniques that can help to make the building of schematics a lot less difficult. Now you have a few more tools in your schematics toolbox to make sure you can build maintainable and highly performing schematics for your projects. Go forth and see what you can create!
