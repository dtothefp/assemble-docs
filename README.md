#### assemble.task
Assemble tasks are very similar to Gulp tasks in that they use a streaming API and can be used to perform a variety of build process steps just as what is typically done with Grunt and Gulp. For our purposes we primarily use Assemble tasks to render templates, although some may be used to process file data outside of a render process. The `assemble.task` method takes:
- task name {String}
- task dependencies {Array} //optional
- callback {Function}
The task pipeline will generally begin with a call to `assemble.src` or the `assemble-push` plugin may be used instead for tasks that need to be loaded in a specific way. Generally, at the end of the pipeline a call to `assemble.dest` will be made to specify the renderable destination. This destination may be omitted for tasks that don't perform renderable actions.
```js
var assemble = require('assemble');
var processDataPlugin = require('./plugins/process-data.js');
 
assemble.task('process-data', function(){
  var start = process.hrtime();

  return assemble.src('**/*.hbs')
    .pipe(processDataPlugin) //the plugin will be passed the `file` object of data associate with each file
    .on('error', function (err) {
       console.log('plugin error', err);
    })
    .on('data', function (fild) {
       console.log(file.path);
    })
    .on('end', function () {
      var end = process.hrtime(start);
      console.log(end);
    }); //nothing is rendered
});
```
More commonly an Assemble task will be used to render `hbs` templates to `html`
```js
var ext = require('gulp-extname');
var path = require('path');
var assemble = require('assemble');
var dest = 'dist';

function makePath(dir) {
  return path.join(process.cwd(), dir);
}

//2nd argument sets up `make-templates` depends on `process-data` assemble task and will not run until it is completed
assemble.task('make-templates', ['process-data'], function(){ 
  var start = process.hrtime();

  return assemble.src(makePath('src/**/*.hbs'))
    .pipe(ext()) //add the .html extension
    .pipe( assemble.dest(makePath(dest)) ) //pass the destination path to `assemble.dest`
    .on('data', function (fild) {
       console.log(file.path);
    })
    .on('end', function () {
      var end = process.hrtime(start);
      console.log(end);
    });
});
```

#### assemble.run
The example tasks above setup the task architecture but will not run until a specific call to `assemble.run` is made. `assemble.run` accepts three arugments:
- task name/s {String|Array}
- task dependencies {Array} //optional
- callback {Function} //called when all tasks are done
Therefore to run the above tasks:
```js
var tasks = ['process-data', 'make-templates'];
 
function done(){
  console.log('tasks completed');
}
 
assemble.run(tasks, done);
 
//or the following will work because `make-templates` depends on `process-data`
 
assemble.run(tasks.pop(), done);
```

#### assemble.watch
Assemble also has it's own watch task built in, which we are using to watch our .hbs and .yml files for changes. A typical watch task could be setup inside of an assemble task or inside of the callback passed to `assemble.run`.
```js
function makeWatch(){
  assemble.watch(['src/**/*.hbs'], ['make-templates']); //when any .hbs in `src` changes run the `make-templates` assemble task (`process-data` will run first because `make-templates` is dependent on it.
  
  assemble.watch(['src/**/*.yml'], ['process-data']); //when any .yml in `src` changes run the `process-data` assemble task
}
 
assemble.task('watch', ['make-templates'], makeWatch);
 
assemble.run(tasks.concat(['watch']));
 
//or alternatively initiate the watch in the callback passed to done
assemble.run(tasks, makeWatch);
```
Another handy use-case to avoid the callback to `assemble.run` might be to make a `done` Assemble task
```js
var done = this.async(); //grunt style callback for ending async process
 
assemble.task('done', ['make-templates'], done);
 
assemble.run(tasks.concat(['watch', 'done'])); //will run all the tasks, start the watch, and exit the async process leaving the watch running.
```

#### assemble.option | assemble.get | assemble.set | assemble.data
Assemble has various getters and setters for storing data on the `assemble.views` and for performing utility functions. One of the most important Assemble options is the `rename` key, which creates a key out of the template file path for which it will be stored on the `assemble.views` object. `assemble.option` is both a getter and a setter so it can be used as follows:
```js
var path = require('path');
 
function cuteKittenPath(fp) {
  return path.join( 'kittens/are/cute', path.basename(fp, path.extname(fp)) ); //removes extension from file and concats with 'kittens/are/cute'
}
 
var ogRenameKey = assemble.option('renameKey'); //this is a getter, returns the rename key function
assemble.option('renameKey', cuteKittenPath);
 
assemble.layouts(['src/layouts/{index,about}.hbs}']);
console.log(Object.keys(assemble.views.layouts)); // ['kittens/are/cute/index', 'kittens/are/cute/about']
 
assemble.option('renameKey', ogRenameKey);
assemble.partials(['src/partials/{coolie,legitssss}.hbs}']);
console.log(Object.keys(assemble.views.layouts)); // ['coolie', 'legitssss']
```

#### It is often useful to set data on the `assemble.cache`. Getters and setters may be used for this purpose as well as `assemble.data`.
```js
assemble.set('coolies', {word: 'to your mother'});
var gotTheCoolies = assemble.get('coolies');
console.log(gotTheCoolies); //{word: 'to your mother'}
gotTheCoolies.dopeness = 'word'; //mutates the object without explicitly calling `assemble.set`
console.log(Object.keys(assemble.get('coolies')); //['word', 'dopeness'];
 
assemble.set('coolies', {sad: 'mutation :-(('});
console.log(assemble.get('coolies')); // {sad: 'mutation :-(('} => overwrite the entire object
```
Data stored on `assemble.data` can be used globally in templates compiled by Assemble.
```js
#src/global_hello.yml
---
hi: "hello"
---
 
#src/global_goodbye.yml
---
goodbye: "ciao"
---
 
#src/something.json
{
  "something": "whatevs"
}
 
assemble.data(['*.yml']);
Object.keys(assemeble.cache.data).forEach(function(key){
  var data = assemeble.cache.data;
  console.log(key, data);  //'global_hello' {hi: 'hello'}, 'global_goodbye' {goodbye: 'ciao'}
});
 
assemble.data(['*.json'], {
  namespace: function(fp){
    return 'dopeness_' + path.basename(fp, path.extname(fp));
  }
});
 
console.log(Object.keys(assemble.cache.data)); //['global_hello', 'global_goodbye', 'dopeness/something']
console.log(assemble.cache.data['dopeness_something']); //{something: 'whatevs'}
 
assemble.data({another: 'prop'}); //this merges rather than overwrites data
var data = assemble.get('data');
console.log(Object.keys(data)); //['global_hello', 'global_goodbye', 'dopeness/something', 'another']
delete data.another; //this mutates the object and sets without explicitly calling `assemble.set`
console.log(Object.keys(assemble.get('data')); //['global_hello', 'global_goodbye', 'dopeness/something']
 
#index.hbs
<h1>{{global_hello.hello}} | {{global_goodbye.goodbye}} | {{dopeness_something.something}}</h1>
 
#rendered index.html
<h1>hello | ciao | whatevs</h1>
```

#### assemble.transform
Transforms are useful for processing data and attaching it to the `assemble` instance as soon as a file containing Assemble tasks is run, and they are run synchronously in the order that they are called (i.e. not in relation to the render cycle specified by `assemble.task`s).  `assemble.transform` expects two arguments but can be passed additional arguments based upon user preference:
- transform name {String}
- callback function that receives arguments from `assemble.transform` with the first argument being the `assemble` instance {Function}
- additional arguments {*}
```js
//tasks/assemble/transforms/my-cool-tranform.js
var path = require('path');
var globby = require('globby');
module.exports = function(assemble, args) {
  var tranformName = args[0];
  var patterns = args[2];
  var root = assemble.get('websiteRoot'); //from config set on the assemble instance
  var cwd = path.join(process.cwd(), root);
  var allNonGlobalDataFiles = globby.sync(patterns, {cwd: cwd});
  assemble.set(transformName, allNonGlobalDataFiles);
};
 
//tasks/assemble/index.js
assemble.set('websiteRoot', 'src');
assemble.transform('nonGlobalFileNames', require('../tranforms/my-cool-tranform'), ['**/*.{yml,yaml,json}', '!**/global_*.{yml,yaml}']);
var nonGlobals = assemble.get('nonGlobalFileNames');
console.log(nonGlobals); //['something.json']
```

#### assemble.middleware
Assemble allows the registration of middleware that will have access to the `file` object at different stages in the render cycle (i.e. onload of templates, as well as pre and post render). Arguments passed to `assemble['onLoad' | 'preRender' | 'postRender"](regexp, callback)`:
- regexp - which files in the pipeline to include/omit
- callback - called with parameters of the `file` data object and a `next` callback
- `assemble.onload` is useful for capturing/processing data outside regardless of a render cycle.
```js
//tasks/assemble/middleware/onload-ex.js
 
var _ = require('lodash');
 
module.exports = function(assemble) {
  return function _getName(collectionName) {
    return function _myOnloadMiddleware(file, next()) {
      var col = assemble.get(collectionName) || {};
      assemble.set(collectionName, _.merge({}, col, file.data); //create a collection on the assemble instance
      next();
    };
  };
};
 
//tasks/assemble/index.js
var assemble = require('assemble');
var onloadCb = require('./middleware/onload-ex')(assemble);
 
assemble.onload(/\/index/, onloadCb('coolies'));
 
assemble.task('process-data', function(){
  var start = process.hrtime();

  return assemble.src('./src/**/*.hbs') //this task does nothing except load data
    .on('error', function (err) {
       console.log('plugin error', err);
    })
    .on('data', function (file) {
       console.log(file.path);
    })
    .on('end', function () {
      var end = process.hrtime(start);
      console.log(end);
    }); //nothing is rendered
});
 
assemble.run(['process-data']);
 
console.log(assemble.get('coolies')); //all data from file paths containing the string 'index'
```
`assemble.preRender` is useful when data being processed is essential immediately prior to render and therefore data added to the `file` object will be available inside of the rendered template context:
```js
//tasks/assemble/middleware/coolies-ex.js
module.exports = function(file, next) {
  file.data.coolies = 'dope';
  next();
};
 
//tasks/assemble/index.js
var ext = require('gulp-extname');
var path = require('path');
var assemble = require('assemble');
var dest = 'dist';
var cooliesMiddleware = require('./middleware/coolies-ex');

function makePath(dir) {
  return path.join(process.cwd(), dir);
}
 
assemble.preRender(/cute\/kittens\/index/, cooliesMiddleware);

assemble.task('render-stuff', function(){ 
  var start = process.hrtime();

  return assemble.src(makePath('src/**/*.hbs'))
    .pipe(ext()) //add the .html extension
    .pipe( assemble.dest(makePath(dest)) )
    .on('data', function (file) {
       console.log(file.path);
    })
    .on('end', function () {
      var end = process.hrtime(start);
      console.log(end);
    });
});
 
assemble.run(['render-stuff']);
 
//src/cute/kittens/index.hbs
<h1>{{coolies}}</h1>
 
//dist/cute/kittens/index.html
<h1>dopeness</h1>
```

#### plugins
Assemble plugins follow the same conventions as the stream based API plugins used in Gulp. Therefore, they are easiest to create using the through2 npm module and just like middleware, give the user access to the `file` object. Plugins can be used in any Assemble task regardless of that task renders files or not. An advantage of plugins is that the `through` api gives access to all `file` objects as they are coming through the pipeline but also provides a callback function as it's second parameter that will called after all `file` objects have been piped through the plugin
```js
//tasks/assemble/plugins/async-stuff.js
var through = require('through2');
var _ = require('lodash');

module.exports = function(assemble, int) {
  var collectedData = {};

  return through.obj(function (file, enc, cb) {
    var data = {};
    data[file.path] = file.data;
    _.merge(collectedData, data);
    cb();
  }, function(cb){
    setTimeout(function(){
      assemble.set('asyncData', collectedData);
      cb();
    }, int);
  });
};
 
//tasks/assemble/index.js
var assemble = require('assemble');
var asyncPlugin = require('./plugins/asyncStuff')(assemble, 1000);

assemble.task('plugin-fun', function(){ 
  var start = process.hrtime();

  return assemble.src(makePath('src/**/*.hbs'))
    .pipe(asyncPlugin)
    .on('data', function (file) {
       console.log(file.path);
    })
    .on('end', function () {
      var end = process.hrtime(start);
      console.log(assemble.get('asyncStuff')); //object of all file data scoped to file path set in the plugin will be available here
    });
});

assemble.run(['plugin-fun']);
```

#### assemble.create
Custom types can be created inside of Assemble, which are very useful for defining custom partials or creating renderable templates that may be loaded in a specific way.
```js
var assemble = require('assemble');
var ogKey = assemble.option('renameKey');
 
function loadModals(patterns) {
  patterns = Array.isArray(patterns) ? patterns : [patterns];
  assemble.options('renameKey', ogKey); //ensure at the time of calling `loadModals` the renameKey will store `modals` on `assemble.views` by filename
  assemble.modals(patterns); //can load the `modal` type using a plural if loading multiple files
};

assemble.create('modal', 'modals', {
  isPartial: true,
  isRenderable: true
});
 
loadModals('./src/components/modals/**/*.hbs');
 
//src/index.hbs
---
page_modals: 
- hello
- goodbye
---
{{#each page_modals}}
  {{modal this}} <!-- `this` is the filename, in this case ./src/components/modals/**/{hello,goodbye}.hbs -->
{{/each}}
```
When creating a `renderable` custom type that will be loaded using `assemble.task` it is useful to use the `assemble-push` module. 
```js
var assemble = require('assemble');
var ext = require('gulp-extname');
var path = require('path');
var push = require('assemble-push')(assemble);


assemble.create('subfolder', {
  isRenderable: true
});
 
assemble.task('load-subfolders', function(){
  assemble.option('renameKey', function(fp){
    var filename = path.basename(fp, path.extname(fp));
    return path.join(path.dirname, filename); //store on `assemble.views.subfolders` by long key name (path plus filename without extension)
  });
 
  assemble.subfolders(['./src/subfolders/**/*.hbs']);
 
  return push('subfolders') //because the subfolders are loaded as a custom type we must use the `assemble-push` module in place of `assemble.src`
    .pipe(ext())
    .pipe(assemble.dest('dist'))
});
```
#### Loaders
Any renderable type in Assemble can be used in combination with a custom loader. This can be very useful for loading type data onto the `assemble.views` in a specific way or creating a desired structure for loaded templates. `assemble.create` can take a second argument which is an array of loaders.
```js
//tasks/assemble/loaders/prepend-coolies.js
module.exports = function(assemble) {
  var prefix = assemble.get('coolies');
  
  return function(templates){
    var keys = Object.keys(templates); //all keys from `assemble.cache[<type that loader is applied to>]
    
    var coolieModals = keys.reduce(function(memo, key){ 
      var data = templates[key];
      key = coolies + '_' + key;
      memo[key] = data;
      return memo;
    }, {});
 
    return coolieModals; //return the object to be stored on `assemble.cache.modals`
  };
};
 
//tasks/assemble/index.js
var assemble = require('assemble');
assemble.set('coolies', 'hotness');
var coolieLoader = require('./loaders/prepend-coolies')(assemble);
 
assemble.create('modal', 'modals', {
  isPartial: true,
  isRenderable: true
}, [coolieLoader]);
 
assemble.modals(['./src/components/modals/{hello,goodbye}.hbs']);
console.log(Object.keys(assemble.cache.modals)); // ['hotness_hello', 'hotness_goodbye']
```
loaders can also be used to load/render files from sources that do not exist (i.e. inherit one directory structure from another)
```js
//tasks/assemble/loaders/inherit.js
var fs = require('fs');
var path = require('path');
var path = require('path');
var globby = require('globby');
var matter = require('gray-matter');
var _ = require('lodash')
 
function lastTwo(fp) {
  var split = path.dirname(fp).split('/');
  var dirname = split[split.length -1];
  var filename = path.basename(fp, path.extname(fp));
  return path.join(dirname, filename);
};

module.exports = function(args) {
  var subfolderSrc = args.src;
  var rootSrc = args.root;
 
  var subfolderFiles = globby.sync(subfolderSrc).map(lastTwo);
  var rootFiles = globby.sync(rootSrc).map(lastTwo);
  var inherit = rootFiles.filter(function(fp){
    return subfolderFiles.indexOf(fp) === -1;
  });
 
  //create an object of all file data that exists in the `src/subfolders` directory
  var subfolderData = subfolderFiles.reduce(function(memo, fp){}, {
    memo[fp] = matter(fs.readFileSync(path.join(process.cwd(), 'src/subfolders', fp + '.hbs'), 'utf8'));
    return memo;
  }, {});
 
  //create an object from all template data in `src/**/*.hbs` excluding the `subfolders` directory and excluding templates in the same directory structure as `src`
  var inheritedData = inherit.reduce(function(memo, fp){}, {
    memo[fp] = matter(fs.readFileSync(path.join(process.cwd(), 'src', fp + '.hbs'), 'utf8'));
    return memo;
  }, {});
 
  return _.merge({}, subfolderData, inheritedData);  //return an object of keys and `file` data that will be stored on `assemble.cache.subfolders`
}
 
//tasks/assemble/index.js
var assemble = require('assemble');
var ext = require('gulp-extname');
var path = require('path');
var push = require('assemble-push')(assemble);
var inheritLoader = require('./loaders/inherit');
 
assemble.create('subfolder', {
  isRenderable: true
}, [inheritLoader]);
 
assemble.task('load-subfolders', function(){
  assemble.option('renameKey', function(fp){
    var filename = path.basename(fp, path.extname(fp));
    return path.join(path.dirname, filename); 
  });
 
  //pass `src` and `root` data to the `inherit` loader
  assemble['subfolder']({
    src: ['src/subfolders/**/*.hbs'],
    root: ['src/**/*.hbs', '!src/subfolders/**/*.hbs']
  });
 
  return push('subfolders')
    .pipe(ext())
    .pipe(assemble.dest('dist'))
});
```

#### assemble.helpers
Helpers can be registered on the `assemble` instance in a two ways:
- passing a globbing pattern to `assemble.helpers`
- passing a name and callback to `assemble.helpers(string, callback)`
Use of helpers in Assemble templates is detailed above HBS Templates Overview. Custom types created using `assemble.create` that are given options of `isPartial: true` register a helper that is the same name as the type as can be seen in the Custom Types example. 
Data on the `assemble` instance context and the `file` data object context can also be accessed within helpers.
```js
assemble.helper('getCoolies', function(str) {
  var app = this.app; //the `assemble` instance
  var coolies = app.get('coolies');
  var fileData = this.context; //the `file.data`
  return [coolies, fileData.coolie_yml_key, str].join('-');
});
```
If using handlebars, subexpressions (i.e. helpers within helpers) can also be used with `isPartial` custom types and registered partials. An example of this can be seen in the [dfoxpowell/build-subfolders](https://github.com/optimizely/marketing-website/blob/dfoxpowell/build-subfolders/website-guts/templates/layouts/seo.hbs#L37) branch of the marketing-website. Subexpressions can be useful for when rendering partials/custom "partial" types the "partial name" string passed to the helper can be manipulated.
```js
assemble.helper('addPrefix', function(filename) {
  var prefix = this.context.prefix; //the `file.data`
  return prefix + '_' + filename;  //will look on `assemble.views[<type or partial name>]` for this prefixed key
});
 
//index.hbs
---
modals:
- coolies
- legits
---
{{#each modals}}
  {{modal (addPrefix this)}} <!-- uses the subexpression (i.e. a helper within a helper) -->
{{/each}}
 
{{> (addPrefix 'dope_partial')}}
```

#### assemble.asyncHelper
Async Helpers can be registered on the `assemble` instance for rendering data that may be returned asynchronously. 
```js
//tasks/assemble/helpers/async/timeout-ex.js
var _ = require('lodash');
 
module.exports = function(type) {
  return function(filenameKey, locals, options, next){
    if (typeof options === 'function') {
      next = options;
      options = locals;
    }
 
    var partial = app.views[type][filenameKey]; //get the renderable data from the `assemble.view` object
    var locs = _.merge({}, this.context, locals); //locals is global data in a rendering template that is passed to the handlebars helper
 
    setTimeout(function(){
      partial.render(locs, function (err, content) {
        if (err) {
          return next(err);
        }
        return next(null, content);
      });
    }, 1000);
  };
};
 
//tasks/assemble/index.js
var asyncHelper = require('./helper/async/timeout');
 
assemble.partials(['.src/components/partials/**/*.hbs']);
assemble.asyncHelper('partials', asyncHelper('partials'));
assemble.create('modal', 'modals', {
  isPartial: true,
  isRenderable: true
});
assemble.asyncHelper('modal', asyncHelper('modals'));
assemble.modals(['./src/components/modals/**/*.hbs']);
 
//src/index.hbs
---
modals:
- coolies
- legits
---
{{#each modals}}
  {{modal this locals}} <!-- rendered using async data -->
{{/each}}
{{partial 'dope_partial' locals}} <!-- `locals` is a global available in a rendering Assemble template -->
```

#### Custom Types, Async Helpers, and Translating `assemble.views[type]`
```js
assemble.asyncHelper('partial', renderTypeHelper('partials'));
function loadPartials () {
  assemble.partials(options.partials, [typeLoader(assemble)]);
}
 
assemble.create('modal', 'modals', {
  isPartial: true,
  isRenderable: true
});
assemble.asyncHelper('modal', renderTypeHelper('modals'));
function loadModals() {
  var modalFiles = config.modals.files[0];
  assemble.modals(normalizeSrc(modalFiles.cwd, modalFiles.src), [typeLoader(assemble)]);
};
 
loadPartials();
loadModals();
 
//in any HBS template
{{> (l10nPartial '<partial_name>')}} <!-- because of partial async helper could also localize using `{{partial '<partial_name>'}}`, potentially less effecient
{{modal '<modal_name>'}} <!-- will automatically be localized by the `assemble.asycHelper(modals...`, potentially not effecient -->

```
Potentially more efficient code for localizing Assemble types:
```js
var assemble = require('assemble');
var typeLoader = require('./loaders/type-loader');
var ogKey = assemble.option('renameKey');
var modalFiles = normalizeSrc(modalFiles.cwd, modalFiles.src);
var partialsPlusModalFiles = options.partials.concat(modalFiles);
 
function loadPartialsAndModals() {
  var currentKey = assemble.option('renameKey'); //cache the current rename key
  assemble.option('renameKey', ogKey); //ensure the default rename key
  assemble.partials(partialsPlusModalFiles, [typeLoader(assemble)]); //load all the partials and modals together
  assemble.option('renameKey', currentKey); //reset rename key to what was current upon initial function execution
}
 
loadPartialsAndModals();
 
//in any HBS template
{{> (l10nPartial '<partial_name>')}} 
{{#each this.layout_modals}}
  {{> (l10nPartial this)}}  <!-- will be localized using Handlebars subexpression by finding keys/data created by the `typeLoader` -->
{{/each}}
```

