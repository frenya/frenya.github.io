---
title: Unit testing conundrum when using webpack
tags: [vscode, tdd, webpack]
style: border
color: primary
comments: true
description: A workaround solution to a known issue
---

## The problem

There's one fundamental flaw when it comes to unit testing while using the suggested webpack configuration for distribution and a plain compilation using `tsc` for testing.

- The `main` attribute of `package.json` must point to `dist/extension.js`
- The code that is loaded by the extension host is therefore `dist/extension.js` even when unit testing
- Any module you reference in your unit tests will be loaded from the `out` folder

This is not a good solution. For instance, you will run into problems in case your tests rely on any of those modules being initialised by the `activate` function (e.g. when you want to reference the extension context).

## The workaround

This is a workaround I'm using at the moment, in case it helps anyone. It uses a global `json` tool to modify `package.json` just for the execution of unit tests. You can get it via `npm install -g json`

Excerpt from my `package.json`
```json
{
  "scripts": {
    "test-compile": "npm run main-out && tsc -p ./",
    "main-out": "json -I -f package.json -e 'this.main=\"./out/extension.js\"'",
    "main-dist": "json -I -f package.json -e 'this.main=\"./dist/extension.js\"'"
  }
}
```

Excerpt from my `launch.json`
```json
{
  "version": "0.2.0",
  "configurations": [{
      "name": "Extension Tests",
      "type": "extensionHost",
      "request": "launch",
      "runtimeExecutable": "${execPath}",
      "args": [
        "--extensionDevelopmentPath=${workspaceFolder}",
        "--extensionTestsPath=${workspaceFolder}/out/test/suite/index",
        "${workspaceFolder}/demo"
      ],
      "outFiles": [
        "${workspaceFolder}/out/test/**/*.js"
      ],
      "preLaunchTask": "npm: test-compile",
      "postDebugTask": "npm: main-dist"
    }
  ]
}
```

## The solution

In the long run VSCode will need to come up with a way to temporarily enforce loading the extension from `out/extension.js` for unit tests.

This has been requested in this [issue on Github](https://github.com/microsoft/vscode/issues/85779){:target="_blank"}