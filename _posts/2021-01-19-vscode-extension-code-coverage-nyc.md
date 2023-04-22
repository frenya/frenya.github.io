---
title: Code coverage of VSCode extensions with NYC
tags: [vscode, tdd, nyc]
style: border
color: primary
comments: true
description: Lessons learned from adding NYC to standard unit testing of vscode extensions
---

*TL;DR: Adding NYC code coverage reporter to vscode extension unit tests can be tricky. You can check my
working test runner [here](https://github.com/frenya/vscode-recall/blob/master/src/test/suite/index.ts){:target="_blank"}.
The article below explains some of the key elements in it.*

## Standard testing flow

When you generate an extension using Yeoman, the following scaffolding will be created:

1. An "Extension Tests" target will be created in launch.json
2. This will start the Extension Host and run a `run` function, typically exported from `${workspaceFolder}/out/test/suite/index`
3. This `run` function takes care of everything, namely
  - setting up a Mocha instance
  - adding all relevant files to its list of tests
  - running mocha and reporting results

All the test run within the Extension Host which is creating a couple of challenges.

## Challenges of adding NYC to the mix

- code covering TypeScript
- running inside the Extension Host
- poor documentation

## NYC internals

NYC is primarily a command line tool. Most tutorials on code covering TypeScript will therefore focus on
the command line options and the content of the `.nycrc` file.

Since we will be running it programatically within the Extension Host, this won't help us and we need to work around that.

Unfortunately, NYC's documentation of the internal workings is virtually non-existent so it took me a bit of
debugging and reverse-engineering to figure out what actually happens and how to translate these command line
instructions into a working TypeScript runner

```
$ cat .nycrc
{
    "extends": "@istanbuljs/nyc-config-typescript",
    // OPTIONAL if you want coverage reported on every file, including those that aren't tested:
    "all": true
}

$ cat test/mocha.opts
--require ts-node/register
--require source-map-support/register
--recursive
<glob for your test files>

$ nyc mocha
```

## Setting up the instance

First of all, we need to create a NYC instance and pass it all the necessary parameters. Also, we need to load some
additional modules needed for TypeScript. The following snippet shows the best way I found to simulate the above command
line options.

```javascript
// Simulates the recommended config option
// extends: "@istanbuljs/nyc-config-typescript",
import * as baseConfig from "@istanbuljs/nyc-config-typescript";

// Recommended modules, loading them here to speed up NYC init
// and minimize risk of race condition
import 'ts-node/register';
import 'source-map-support/register';

export async function run(): Promise<void> {
	const testsRoot = path.resolve(__dirname, '..');

  // Setup coverage pre-test, including post-test hook to report
  const nyc = new NYC({
    ...baseConfig,
    cwd: path.join(__dirname, '..', '..', '..'),
    reporter: ['text-summary', 'html'],
    all: true,
    silent: false,
    instrument: true,
    hookRequire: true,
    hookRunInContext: true,
    hookRunInThisContext: true,
    include: [ "out/**/*.js" ],
    exclude: [ "out/test/**" ],
  });
  await nyc.wrap();
...
```

Notice, that we are actully instrumenting the files in the `out` directory - they will be translated to your `src/**/*.ts` files using source maps.

If you want to check your include/exclude configuration, you can run the following:

```javascript
// Debug which files will be included/excluded
console.log('Glob verification', await nyc.exclude.glob(nyc.cwd));
```

You should see all the relevant files in the console. If not, tweak the include/exclude parameters until you do.

## Instrumenting the JavaScript

First thing that needs to happen in order for the code coverage reporting to work is so called "instrumenting".
NYC does this on the fly by replacing the `require` function with an instrumented version of it. This is done
when the `nyc.wrap()` function is called (see last line of the snippet above). Any module loaded before this
call is not instrumented and won't appear in your coverage report.

I am using the following safety check to detect any modules loaded prior to the `nyc.wrap()` call
and re-requiring them:

```javascript
// Print a warning for any module that should be instrumented and is already loaded,
// delete its cache entry and re-require
// NOTE: This would not be a good practice for production code (possible memory leaks), but can be accepted for unit tests
// NOTE: nyc.exclude handles both the include and exlude patterns, the name is a bit misleading here
Object.keys(require.cache).filter(f => nyc.exclude.shouldInstrument(f)).forEach(m => {
  console.warn('Module loaded before NYC, invalidating:', m);
  delete require.cache[m];
  require(m);
});
```

[This issue](https://gitlab.com/gitlab-org/gitlab-vscode-extension/-/issues/224){:target="_blank"} on GitLab has a few pointers in that respect.

## Reporting coverage

This is another challenge you will face when running in the Extension host. You can see I am using the
'text-summary' and 'html' reporters. The HTML files were created as expected, but I couldn't get the text
summary reporter to work. This was due to the fact that NYC reports the results directly into `process.stdout`
stream and they don't show in the debug console. The following function did the trick:

```javascript
async function captureStdout(fn) {
  let w = process.stdout.write, buffer = '';
  process.stdout.write = (s) => {
    buffer = buffer + s;
    return true; 
  };
  await fn();
  process.stdout.write = w;
  return buffer;
}
```

I then run the reporter like this

```javascript
// Capture text-summary reporter's output and log it in console
console.log(await captureStdout(nyc.report.bind(nyc)));
```

## Putting it all together

You can have a look at the full file in one of my projects [here](https://github.com/frenya/vscode-recall/blob/master/src/test/suite/index.ts){:target="_blank"}.

