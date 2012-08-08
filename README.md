# webpack

As developer you want to reuse existing code.
As with node.js and web all files are already in the same language, but it is extra work to use your code with the node.js module system and the browser.

The goal of `webpack` is to bundle CommonJs (and AMD) modules into javascript files which can be loaded by `<script>`-tags.
Simply concatenating all required files has a disadvantage: many code to download (and execute) on page load.
Therefore `webpack` uses the `require.ensure` function ([CommonJs/Modules/Async/A](http://wiki.commonjs.org/wiki/Modules/Async/A)) to split your code automatically into multiple bundles which are loaded on demand.
This happens mostly transparent to the developer.
Dependencies are resolved for you.
The result is a smaller initial code download which results in faster page load.

Another common thing in web development is that some files have to be preprocessed before send to the client (ex. template files).
This introduce more complexity to the compile step.
`webpack` supports loaders which process files before including them.
You as developer can use such files like any other module.

[![build status](https://secure.travis-ci.org/webpack/webpack.png)](http://travis-ci.org/webpack/webpack)

**TL;DR**

* bundle [CommonJs](/webpack/webpack/tree/master/examples/commonjs/) and/or [AMD](/webpack/webpack/tree/master/examples/mixed/) modules for browser
* reuse server-side code (node.js) on client-side
* create multiple files which are loaded on demand (faster page load in big webapps or on mobile connections)
* dependencies managed for you, on compile time (no resolution on runtime needed)
* loaders can preprocess files

## Quick start guide

``` javascript
var moduleA = require("module/file");
var moduleB = require("./relativeFile");
var moduleC = require("../stuff/cup.coffee");
function getTemplate(name) {
	return require("./templates/" + name + ".jade");
}
require("bootstrap/less/bootstrap.less");
```

``` shell
npm install webpack -g
webpack lib/yourEntryModule.js output/bundle.js
```

## Goals

* make node.js and browser development similar
* minimize code size (mobile connection)
 * minimize code size on initial download
 * download code only on demand
* require minimal configuration, but offer a maximum
 * load polyfills for node-specific things if used
 * offer replacements for node buildin libraries
* support [npm](https://npmjs.org/) and [jam](http://jamjs.org/)

# Examples

See [example webapp](http://webpack.github.com/example-app/).

More [examples](https://github.com/webpack/webpack/tree/master/examples).

## Simple Example

``` javascript
// a.js
var b = require("./b");
b.stuff("It works");

// b.js
exports.stuff = function(text) {
	console.log(text);
}
```

are compiled to (reformatted)

``` javascript
(/* small webpack header */)
({
0: function(module, exports, require) {

	var b = require(1);
	b.stuff("It works");

},
1: function(module, exports, require) {

	exports.stuff = function(text) {
		console.log(text)
	}

}
})
```

## Code Splitting

### Example

``` javascript
var a = require("a");
var b = require("b");
require.ensure(["c"], function(require) {
	require("b").xyz();
	var d = require("d");
});
```

```
File 1: web.js
- code of that file
- code of module a and dependencies
- code of module b and dependencies

File 2: 1.web.js
- code of module c and dependencies (but code is not used)
- code of module d and dependencies
```

Initially only `web.js` is included (and loaded) by your application.
`1.web.js` is loaded when the call to `require.ensure` happens.
After the second bundle (`1.web.js`) is loaded, the callback function will be invoked.

See [details](/webpack/webpack/tree/master/examples/code-splitting) for exact output.

See [more examples](/webpack/webpack/tree/master/examples).

## Reusing node.js code

`webpack` was built to support most of the code that was coded for node.js environment.
For example this works out of the box:

* `require("./templates/" + templateName);`
* `require(condition ? "moduleA" : condition2 ? "moduleB" : "./localStuff");`
* `function xyz(require) { require("text"); } xyz(function(a) { console.log(a) });`
* `var r = require; r("./file");` with warning
* `function xyz(require) { require("./file"); } xyz(require);` with warning
* `try { require("missingModule"); } catch(e) { console.log("missing") }` with warning
* `var require = function(a) { console.log(a) }; require("text");`
* `if(condition) require("optionalModule")` with warning if missing

## Browser replacements

Somethings it happens that browsers require other code than node.js do.
`webpack` allow module developers to specify replacements which are used in the compile process of `webpack`.

Modules in `web_modules` and `jam` replace modules in `node_modules`.
`filename.web.js` replaces `filename.js` when required without file extension.

in options: `alias: { "http": "http-browserify" }`

in shell: `--alias http=http-browserify`

## Contexts

If the required module is not known while compile time we get into a problem.
A solution is the method `require.context` which takes a directory as parameter
and returns a function which behaves like the `require` function issued from a file
in this directory (but only if used for files in that directory).
The whole directory is included while compiling, so you have access to all files in it.

### Example

We have a directory full of templates, which are compiled javascript files.
A template should be loaded by template name.

``` javascript
var requireTemplate = require.context("./templates");
function getTemplate(templateName) {
	return requireTemplate("./" + templateName);
}
```

In addition to that `webpack` uses the `require.context` function automatically
if you use variables or other not parse-able things in the `require` function.
That means the following code behaves like the above:

``` javascript
function getTemplate(templateName) {
	return require("./templates/" + templateName);
}
// is compiled like: return require.context("./templates")("./"+templateName)
// which compiles to: return require(123)("./"+templateName)
```

See [details](/webpack/webpack/tree/master/examples/require.context) for complete example.

When try to store the `require` function in another variable or try to pass it as parameter,
`webpack` convert it to a `require.context(".")` to be compatible.
There is a warning emitted in this case.

*Warning: The complete code in the directory are included. So use it carefully.*

## Loaders

You can use a syntax for loader plugins to preprocess files before emitting javascript code to the bundle.

The following example loads the raw content of a file with the `raw` loader:

``` javascript
var content = require("raw!./file.txt");
```

Multiple loader plugins can be prepended by separating them with `!`.
The loader plugins are resolved like in normal `require` call but with different default extension.

The `raw` loader plugin is looked up at modules `raw-webpack-web-loader`, `raw-webpack-loader`, `raw-web-loader`, `raw-loader`, `raw`
and the following files are looked up: `index.webpack-web-loader.js`, `index.webpack-loader.js`, `index.web-loader.js`, `index.loader.js`, `index`, `index.js`.
Note that the `web-` versions are omitted if loaders are used in node.js.

See [example](/webpack/webpack/tree/master/examples/loader).

The following loaders are included in webpack:

* [`raw`](https://github.com/webpack/raw-loader): Loads raw content of a file (as utf-8)
* [`json`](https://github.com/webpack/json-loader) (default at `.json`): Loads file as JSON
* [`jade`](https://github.com/webpack/jade-loader) (default at `.jade`): Loads jade template and returns a function
* [`coffee`](https://github.com/webpack/coffee-loader) (default at `.coffee`): Loads coffee-script like javascript
* [`css`](https://github.com/webpack/css-loader): Loads css file with resolved imports and returns css code
* [`less`](https://github.com/webpack/less-loader): Loads and compiles a less file and returns css code
* [`val`](https://github.com/webpack/val-loader): Excutes code as module and consider exports as javascript code
* [`bundle`](https://github.com/webpack/bundle-loader): Wraps request in a `require.ensure` block
* [`file`](https://github.com/webpack/file-loader): Emits the file into the output folder and returns the (relative) url (`file/{ext}` for some extensions)
* [`style`](https://github.com/webpack/style-loader): Adds result of javascript execution to DOM
* [`script`](https://github.com/webpack/script-loader): Executes a javascript file once in global context (like in script tag), requires are not parsed. Use this to include a library. ex. `require("script!./jquery.min.js")`. This is synchron, so the `$` variable is available after require.
* (`.css` defaults to `style!css` loader, so all css rules are added to DOM)
* (`.less` defaults to `style!css!val/cacheable!less` loader, so all less rules are added to DOM)

See docs for loader in github repo of the loader.

## TL;DR

``` javascript
var a = require("a"); // require modules
var b = require("./b"); // and files
                          // like in node.js

// polyfill require method to use the new members in node.js too
require = require("webpack/require-polyfill")(require.valueOf());

// create a lazy loaded bundle
require.ensure([], function(require) {
	var c = require("c");

	// require json
	var packageJson = require("../package.json");

	// or jade templates, coffee-script, and many more with own loaders
	var result = require("./template.jade")(require("./dataFrom.coffee"));

	// files are compiled to javascript and packed into the bundle...
});
```

... and compile from the shell with:

```
webpack lib/input.js js/output.js
```

try `--min` to minimize with `uglify-js`.

## Limitations

### `require`-function

* `require` should not be overwritten, except from polyfill
* `require.ensure` should not be overwritten or called indirect
* `require.context` should not be overwritten or called indirect
* the argument to `require.context` should be a literal or addition of multiple literals
* An indirect call of `require` should access a file in current directory: This throws an exception: `var r = require; r("../file");`

The following cases could result in **too much code** in result file if used wrong:

* indirect call of `require`: `var r = require; r("./file");`. It includes the whole directory.
* `require.context`. It includes the whole directory.
* expressions in require arguments: `require(variable)`. It includes the whole directory. (except from `?:`-operator `require(condition ? "a" : "b")`)
* the function passed to `require.ensure` is not inlined in the call. Only requires in inlined function move into the second bundle.


### node.js specific modules

As node.js specific modules like `fs` will not work in browser they are not included (by default) and cause an exception.
You should replace them by own modules if you want to use them.
For some simple modules are replacements included in `webpack`.
Expensive replacements are not needed by everyone, so that are not included by default.
You need to specify `--alias [module]=[replacement]` to use them.
A warning saying that some module is missing is emitted in the case you use it without providing a replacement.

Some credit goes to the browserify contributors, you can use replacements provided by them.

Included simple replacements:

* `assert`: copy of node.js' version, small change
* `buffer`: copy of node-browserify's version
* `buffer_ieee754`: copy of node-browserify's version
* `child_process`: disabled
* `events`: copy of node.js' version
* `path`: copy of node.js' version
* `punycode`: copy of node.js' version, one line removed (http://mths.be/punycode by @mathias)
* `querystring`: copy of node.js' version
* `string_decoder`: copy of node.js' version
* `url`: copy of node.js' version
* `util`: copy of node.js' version

Here is a list of possible useful replacements: (intentionally not by default)

* `http=http-browserify`
* `vm=vm-browserify`
* TODO provide some more replacements

## Usage

### Shell

`webpack` offers a command line interface:

after `npm install webpack -g` you can use the `webpack` command

if invoked without arguments it prints a usage:

```
Usage: webpack <options> <input> <output>

Options:
  --min                Minimize it with uglifyjs                                [boolean]  [default: false]
  --filenames          Output Filenames Into File                               [boolean]  [default: false]
  --options            Options JSON File                                        [string]
  --public-prefix      Path Prefix For JavaScript Loading                       [string]
  --libary             Stores the exports into this variable                    [string]
  --colors             Output Stats with colors                                 [boolean]  [default: false]
  --single             Disable lazy loading                                     [boolean]  [default: false]
  --json               Output Stats as JSON                                     [boolean]  [default: false]
  --by-size            Sort modules by size in Stats                            [boolean]  [default: false]
  --verbose            Output dependencies in Stats                             [boolean]  [default: false]
  --alias              Set a alias name for a module. ex. http=http-browserify  [string]
  --debug              Prints debug info to output files                        [boolean]  [default: false]
  --watch              Recompiles on changes (except loaders)                   [boolean]  [default: false]
  --watch-delay        Timeout to wait for the last change                      [string]
  --progress           Displays a progress while compiling                      [boolean]  [default: false]
```

### Programmatically Usage

``` javascript
webpack(context, moduleName, [options], callback)
webpack(absoluteModulePath, [options], callback)
```

#### `options`

You can also save this options object in a JSON file and use it with the shell command.

``` javascript
{
 output: "out/file.js", // required
 // output file to initial chunk

 outputDirectory: "out/dir", // default: extract directory from output
 // output directory for file

 outputPostfix: ".chunk.js", // default: "." + output
 // postfix appended to id of lazy loaded chunks

 context: "/home/node/stuff",
 // default: [context] parameter if Programmatically Usage
 // default: process.cwd() if Shell Usage
 // paths in stats and debug sourceUrl are shortened to this base directory

 ouputJsonpFunction: "myJsonp", // default: "webpackJsonp"
 // jsonp function used for lazy loaded chunks,
 // should be unique for all instances of webpack in a page

 publicPrefix: "http://static.url.com/asserts/", // default: ""
 // path to create the chunks url relative to page
 // deprecated name: scriptSrcPrefix

 libary: "mylib", // default: null
 // store the exports of the entrace module in a variable of this name
 // use this to create a libary from webpack

 includeFilenames: true, // default: false
 // include the filename of each module as comment before the module

 single: false, // default: false
 // ignore all Code Splitting and emit only a single file
 // all code is included and should work as with Code Splitting

 debug: true, // default: false
 // put the source of the modules into annotated eval,
 // which cause a nice debug experience in some dev tools

 watch: true, // default: false
 // recompiles on changes on module and contexts (currently not on loaders)
 // unchanged files are cached for greater performance

 watchDelay: 1000, // default: 200
 // delay in ms before recompile after the last file change

 events: new EventEmitter(), // default: new EventEmitter()
 // EventEmitter on which events for the compile process are fired
 // events:
 //  -- bundling process --
 //  "bundle"           (stats) the bundle is finished
 //  "bundle-invalid"   () fired when the bundle gets invalid
 //         [bundle-invalid is only fired in watch mode]
 //  "start-writing"    (hash) fired when webpack starts writing
 //  -- events for dependencies --
 //  "module"           (module, filename) before a module is loaded
 //  "context"          (module, dirname) before a context is loaded
 //  "dependency"       (filename) before a dependency is loaded
 //  "loader"           (filename) before a loader is required
 //  -- events for progress --
 //  "task"             (name?) start of a task
 //  "task-end"         (name?) end of a task

 noWrite: true, // default: undefined
 // if true webpack do not write out any file

 parse: {
  // options for parsing

  overwrites: {
   "myglobal": "modulename-of-myglobal"
   // defaults: (defaults are also included if you define your own)
   // process: "__webpack_process",
   // module: "__webpack_module",
   // console: "__webpack_console",
   // global: "__webpack_global",
   // Buffer: "buffer+.Buffer" // -> require("buffer").Buffer
   // "__dirname": "__webpack_dirname",
   // "__filename": "__webpack_filename"
  },
  // inject a free variable named "myglobal" which are required as
  // require("modulename-of-myglobal")
  // to each module which uses "myglobal"
 }

 resolve: {
  // options for resolving

  paths: ["/my/absolute/dirname"],
  // default: (defaults are also included if you define your own)
  //   [".../buildin",
  //     ".../buildin/web_modules", ".../buildin/name_modules",
  //     ".../node_modules"]
  // search paths for modules

  modulesDirectorys: ["xyz_modules", "node_modules"],
  // default: (defaults are NOT included if you define your own)
  //  ["web_modules", "jam", "node_modules"];
  // directories to be searched for modules

  alias: {
   "old-module": "new-module"
  },
  // replace a module

  extensions: ["", ".www.js", ".js"],
  // defaults: (defaults are NOT included if you define your own)
  //   ["", ".webpack.js", ".web.js", ".js"]
  // postfixes for files to try

  loaderExtensions: [".loader.js", ".www-loader.js", "", ".js"],
  // defaults: (defaults are NOT included if you define your own)
  //   [".webpack-web-loader.js", ".webpack-loader.js",
  //      ".web-loader.js", ".loader.js", "", ".js"]
  // postfixes for loaders to try

  loaderPostfixes: ["-loader", "-xyz", ""],
  // defaults: (defaults are NOT included if you define your own)
  //   ["-webpack-web-loader", "-webpack-loader",
  //      "-web-loader", "-loader", ""]
  // postfixes for loader modules to try

  loaders: [{test: /\.generator.js/, loader: "val"}],
  // default: (defaults are also included if you define your own)
  //   [{test: /\.coffee$/, loader: "coffee"},
  //    {test: /\.json$/, loader: "json"},
  //    {test: /\.jade$/, loader: "jade"},
  //    {test: /\.css$/, loader: "style!css"},
  //    {test: /\.less$/, loader: "style!css!val!less"}]
  // automatically use loaders if filename match RegExp
  // and no loader is specified

  postprocess: {
   normal: [function(filename, callback) {
    // webpack will not find files including ".exclude."
    if(/\.exclude\.[^\\\/]*$/.test(filename))
	 return callback(new Error("File is excluded"));
	callback(null, filename);
   }],
   // defaults: []
   // postprocess resolved filenames by all specified async functions
   // a postprocessor must call the callback

   context: [],
   // same as postprocess.normal but for contextes
  }
 }

 postLoaders: [{test: /\.export.js$/, loader: "export"}],
 // default: []
 // syntax like resolve.loaders
 // all loaders which matches the file are applied after the
 // normal loaders. This cannot be overridden in the require call.

 preLoaders: [{test: /\.txt$|\.html$/, loader: "normalizeNLs"}],
 // default: []
 // syntax like resolve.loaders
 // all loaders which matches the file are applied before the
 // normal loaders. This cannot be overridden in the require call.
}
```

#### `callback`

`function(err, source / stats)`
`source` if `options.output` is not set (DEPRECATED)
else `stats` as json:

``` javascript
{
 hash: "52bd9213...38d",
 time: 1234, // in ms
 chunkCount: 2,
 modulesCount: 10,
 modulesIncludingDuplicates: 10,
 modulesFirstChunk: 3,
 fileSizes: {
  "output.js": 1234,
  "1.output.js": 2345
 },
 warnings: [ "Some warning" ],
 errors: [ "Some error" ],
 fileModules: {
  "output.js": [
   { id: 0, size: 123, filename: "/home/.../main.js", reasons: [
    { type: "main" }
   ]},
   { id: 1, size: 234, filename: "...", reasons: [
    { type: "require", // or "context", "async require", "async context"
	  count: 2,
	  filename: "/home/.../main.js",
	  // or dirname: "..." // for type = "context" or "async context"
	}
   ]},
   ...
  ],
  "1.output.js": [...]
 },
 subStats: [...], // stats for embedded webpacks
}
```

### with grunt

see [grunt-webpack](https://github.com/webpack/grunt-webpack).

## Bonus features

### File hash

You can use `[hash]` in `publicPrefix`, `output`, `outputDirectory`, `outputPostfix` and in the shell parameters.
`webpack` will replace it with a hash of your files, when writing.

### From shell

Combine the options `--colors --watch --progress` to get a pretty shell compilation.

## Comparison

<table>
 <tr>
  <th>
	Feature
  </th>
  <th>
	webpack/<br/>webpack
  </th>
  <th>
	medikoo/<br/>modules-<br/>webmake
  </th>
  <th>
	substack/<br/>node-<br/>browserify
  </th>
 </tr>

 <tr>
  <td>
	single bundle
  </td>
  <td>
	yes
  </td>
  <td>
	yes
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	multiple bundles, Code Splitting
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	indirect require
	<code>var r = require; r("./file");</code>
  </td>
  <td>
	in directory
  </td>
  <td>
	include by config option
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	concat in require
	<code>require("./fi" + "le")</code>
  </td>
  <td>
	yes
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	variables in require (local)
	<code>require("./templates/"+template)</code>
  </td>
  <td>
	yes, complete directory included
  </td>
  <td>
	include by config option
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	variables in require (global)
	<code>require(moduleName)</code>
  </td>
  <td>
	no
  </td>
  <td>
	include by config option
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	node buildin libs
	<code>require("http");</code>
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	<code>process</code> polyfill
  </td>
  <td>
	yes, on demand
  </td>
  <td>
	no
  </td>
  <td>
	yes, ever
  </td>
 </tr>

 <tr>
  <td>
	<code>module</code> polyfill
  </td>
  <td>
	yes, on demand
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	<code>require.resolve</code>
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	<code>global</code> to <code>window</code> mapping
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	requirable files
  </td>
  <td>
	filesystem
  </td>
  <td>
	directory scope
  </td>
  <td>
	filesystem
  </td>
 </tr>

 <tr>
  <td>
	different modules with same name
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	eliminate duplicate code
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	require JSON
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	plugins
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	loaders
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
 </tr>

 <tr>
  <td>
	compile coffee script
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	watch mode
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	debug mode
  </td>
  <td>
	yes
  </td>
  <td>
	no
  </td>
  <td>
	yes
  </td>
 </tr>

 <tr>
  <td>
	libaries
  </td>
  <td>
	on global obj
  </td>
  <td>
	no
  </td>
  <td>
	requirable
  </td>
 </tr>

 <tr>
  <td>
	browser replacements
  </td>
  <td>
	<code>web_modules</code> and <code>.web.js</code>
  </td>
  <td>
	no
  </td>
  <td>
	by alias config option
  </td>
 </tr>

 <tr>
  <td>
	compiles with (optional) modules missing
  </td>
  <td>
	yes, emit warnings
  </td>
  <td>
	no
  </td>
  <td>
	no
  </td>
 </tr>
</table>


## Tests

You can run the unit tests with `npm test`.

You can run the browser tests:

```
cd test/browsertests
node build
```

and open `test.html` in browser. There must be several OKs in the file, no FAIL and no RED boxes.

## Contribution

You are welcome to contribute by writing issues or pull requests.
It would be nice if you open source your own loaders or webmodules. :)

You are also welcome to correct any spelling mistakes or any language issues, because my english is not perfect...

## Future plans

* more polyfills for node.js buildin modules, but optional
* split require-polyfill from webpack and offer it as seperate module
* cache in folder and allow reuseing it
* migrate tests to mocha
* write it into the wiki if you have more ideas...

## License

Copyright (c) 2012 Tobias Koppers

MIT (http://www.opensource.org/licenses/mit-license.php)

## Dependencies

* [esprima](http://esprima.org/)
* [optimist](https://github.com/substack/node-optimist)
* [uglify-js](https://github.com/mishoo/UglifyJS)
* [sprintf](https://github.com/maritz/node-sprintf)