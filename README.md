# karma-ng-html2js-preprocessor

[![js-standard-style](https://img.shields.io/badge/code%20style-standard-brightgreen.svg?style=flat-square)](https://github.com/karma-runner/karma-ng-html2js-preprocessor)
 [![npm version](https://img.shields.io/npm/v/karma-ng-html2js-preprocessor.svg?style=flat-square)](https://www.npmjs.com/package/karma-ng-html2js-preprocessor) [![npm downloads](https://img.shields.io/npm/dm/karma-ng-html2js-preprocessor.svg?style=flat-square)](https://www.npmjs.com/package/karma-ng-html2js-preprocessor)

[![Build Status](https://img.shields.io/travis/karma-runner/karma-ng-html2js-preprocessor/master.svg?style=flat-square)](https://travis-ci.org/karma-runner/karma-ng-html2js-preprocessor) [![Dependency Status](https://img.shields.io/david/karma-runner/karma-ng-html2js-preprocessor.svg?style=flat-square)](https://david-dm.org/karma-runner/karma-ng-html2js-preprocessor) [![devDependency Status](https://img.shields.io/david/dev/karma-runner/karma-ng-html2js-preprocessor.svg?style=flat-square)](https://david-dm.org/karma-runner/karma-ng-html2js-preprocessor#info=devDependencies)

> Preprocessor for converting PHP files to [AngularJS 1.x](http://angularjs.org/) and [Angular 2](http://angular.io/) templates.

## Rationale
Some people (like me) like to use PHP to write server side stuff. While PHP
admittedly isn't the prettiest language in town (although it's come a long way),
it _does_ come with a gazillion features that make writing HTML easier. Also, a
lot of existing code and libraries are simply in PHP.

When using PHP to generate AngularJS frontside code, it happens that your HTML
is generated by PHP and only afterwards used as an Angular template. This can be
convenient when some logic needs to be prerendered (e.g. for search engines).
This little module allows you to use these templates in your unit tests.

`karma-ng-php2js-preprocessor` was based on
[`karma-ng-html2js-preprocessor`](https://github.com/karma-runner/karma-ng-html2js-preprocessor)
and shares most of its options/functionality.

## Installation
The easiest way is to keep `karma-ng-php2js-preprocessor` as a devDependency in your `package.json`. Just run

```bash
$ npm install karma-ng-php2js-preprocessor --save-dev
```

## Configuration
```js
// karma.conf.js
module.exports = function(config) {
  config.set({
    preprocessors: {
      '**/*.php': ['ng-php2js']
    },

    files: [
      '*.js',
      '*.php',
      '*.php.ext',
      // if you want to load template files in nested directories, you must use this
      '**/*.php'
    ],

    // if you have defined plugins explicitly, add karma-ng-html2js-preprocessor
    // plugins: [
    //     <your plugins>
    //     'karma-ng-php2js-preprocessor',
    // ]

    ngPhp2JsPreprocessor: {
      // strip this from the file path
      stripPrefix: 'public/',
      stripSuffix: '.ext',
      // prepend this to the
      prependPrefix: 'served/',

      // or define a custom transform function
      // - cacheId returned is used to load template
      //   module(cacheId) will return template at filepath
      cacheIdFromPath: function(filepath) {
        // example strips 'public/' from anywhere in the path
        // module(app/templates/template.html) => app/public/templates/template.html
        var cacheId = filepath.strip('public/', '');
        return cacheId;
      },

      // - setting this option will create only a single module that contains templates
      //   from all the files, so you can load them all with module('foo')
      // - you may provide a function(htmlPath, originalPath) instead of a string
      //   if you'd like to generate modules dynamically
      //   htmlPath is a originalPath stripped and/or prepended
      //   with all provided suffixes and prefixes
      moduleName: 'foo',

      // Path to PHP binary. Defaults to `'/user/bin/php'`.
      phpBin: '/usr/bin/php'
    }
  })
}
```

### Multiple module names
Use *function* if more than one module that contains templates is required.

```js
// karma.conf.js
module.exports = function(config) {
  config.set({
    // ...

    ngHtml2JsPreprocessor: {
      // ...

      moduleName: function (htmlPath, originalPath) {
        return htmlPath.split('/')[0];
      }
    }
  })
}
```

If only some of the templates should be placed in the modules,
return `''`, `null` or `undefined` for those which should not.

```js
// karma.conf.js
module.exports = function(config) {
  config.set({
    // ...

    ngHtml2JsPreprocessor: {
      // ...

      moduleName: function (htmlPath, originalPath) {
        var module = htmlPath.split('/')[0];
        return module !== 'tpl' ? module : null;
      }
    }
  })
}
```

## How does it work ?
This preprocessor runs PHP files against the PHP parser and converts the
resulting HTML to AngularJS modules like [this plugin it was based
on](https://github.com/karma-runner/karma-ng-html2js-preprocessor). These
modules, when loaded, put these HTML files into the `$templateCache` and
therefore Angular won't try to fetch them from the server.

For instance this `template.php`...
```php
<?php $aVariable = 'foo' ?>
<div><?php echo $aVariable ?></div>
```
... will be served as `template.php.js`:
```js
angular.module('template.php', []).run(function($templateCache) {
  $templateCache.put('template.php', '<div>foo</div>')
})
```

----

## Angular2 template caching
For using this preprocessor with Angular 2 templates use `angular: 2` option ini
the config file.

```js
// karma.conf.js
module.exports = function(config) {
  config.set({
    // ...

    ngHtml2JsPreprocessor: {
      // ...

      angular: 2
    }
  })
}
```

The template `template.php`...
```php
<?php $aVariable = 'foo' ?>
<div><?php echo $aVariable ?></div>
```
... will be served as `template.php.js` that sets the template content in the
global $templateCache variable:
```js
window.$templateCache = window.$templateCache || {}
window.$templateCache['template.php'] = '<div>foo</div>';
```

To use the cached templates in your Angular 2 tests use the provider for the
Cached XHR implementation - `CACHED_TEMPLATE_PROVIDER` from
`angular2/platform/testing/browser`. The following shows the change in
`karma-test-shim.js` to use the cached XHR and template cache in all your tests.
```js
// karma-test-shim.js
...
System.import('angular2/testing').then(function(testing) {
  return System.import('angular2/platform/testing/browser').then(function(providers) {
    testing.setBaseTestProviders(
      providers.TEST_BROWSER_PLATFORM_PROVIDERS,
      [providers.TEST_BROWSER_APPLICATION_PROVIDERS, providers.CACHED_TEMPLATE_PROVIDER]);
  });
}).then(function() {
...
```

Now when your component under test uses `template.php` in its `templateUrl` the
contents of the template will be used from the template cache instead of making
an XHR to fetch the contents of the template. This can be useful while writing
fakeAsync tests where the component can be loaded synchronously without the need
to make a XHR to get the templates.

## Gotchas
Like any preprocessor the module internally receives the contents of the PHP
file being parsed. Needless to say, these are simply discarded.

The preprocessor will be mostly useful during testing, since obviously for your
actual project you'll use PHP to generate the templates. Having said that, in
special cases it might come in handy in production.

---

For more information on Karma see the [homepage].


[homepage]: http://karma-runner.github.com
