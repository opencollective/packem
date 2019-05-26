# The Plugin System

This section outlines the various things needed to get you started with writing efficient Packem plugins. You **must** go through this section well enough to be able to build a Packem plugin. The examples are in TypeScript just so to keep an extra hand at the proper type a variadic deserves. If there are any issues or problems with this system, please file an issue or submit a PR.

Packem plugins are in-memory classes that are instantiated during build time. An instantiated plugin does absolutely nothing to the bundle cycle so therefore **event hooks** are needed to allow such manipulations.

## The `PackemPlugin` class

The `PackemPlugin` class is an abstract class that stands as a container for newly created plugins. Plugins need to extend `PackemPlugin` to be able to use the event hooks.

As an example plugin, let's create a plugin that logs '🐶' to the console everytime a JavaScript file is added to the module graph. Let's start off with creating a class `PuppyPlugin`.

```javascript
const { PackemPlugin } = require("packem");

// This does nothing! 😭
class PuppyPlugin extends PackemPlugin {}

module.exports = PuppyPlugin;
```

## Using Event Hooks to manipulate the bundle cycle

To put more sense into our plugin, we'll need an event that _allows us to check if a JavaScript module is added to the module graph_. The event `onModuleBundle` does exactly that. Event hooks are defined as class methods that might or might not take any arguments depending on the event type. In this case, `onModuleBundle` takes the _module interface_ (more on this later) which allows us to read or write to the module graph.

```javascript
const { PackemPlugin } = require("packem");

// This does nothing!
class PuppyPlugin extends PackemPlugin {
  onModuleBundle(mod) {
    if (mod.extension == "js") {
      console.log("You bet I have a cute 🐶!");
    }
  }
}

module.exports = PuppyPlugin;
```

## Adding a plugin to `packem.config.yml`

One last step to avail your plugin to a project is adding it to the configuration file. In the `packem.config.yml`, add the following then run the command `packem` in the terminal:

```yml
root: ./src/index.js
output: ./dist
plugins:
  packem-puppy-plugin: on
```

Depending on the number of JavaScript files in your project, you should've ended up with some logs in your terminal (`"You bet I have a cute 🐶!"`). Try adding more JavaScript files, rerun `packem` and see what happens.

# Advanced Plugin APIs

Now that you're able to create a basic plugin without understanding much whats going on, go through this section to be able to create more intense Packem plugins.

## Plugin Rules

There are a few procedures that one needs to take into account to **create**, **publish** and **use** a plugin(s):

- If your plugin is on the `npm` registry as a package, it must be in the form `packem-<name>-plugin` where `<name>` (without the angle brackets) is your plugin name.
- Your package must export **ONLY ONE** class that extends `PackemPlugin`.
- All plugins fall under the `plugins` field in the config, relative to `root`, `output` and `transformer`.
- If a plugin is defined in the config _and is listed to be a common plugin_, it must follow the format `<name>-plugin`. A common plugin is any plugin under the packem scope on npm like `@packem/dev-plugin`, `@packem/file-plugin`. You can find more about common plugins [here](https://github.com/packem/packem/blob/master/docs/common-plugins.md).
- If a plugin is defined in the config _and is custom_, it must follow the format `packem-<name>-plugin`. Packem can pick this from your plugin's `package.json`. This means your plugin's class doesn't need to follow any format so this is fine.

    ```module.exports = class FunkyApple extends PackemPlugin {}```

- A correctly defined plugin has access to any option(s) passed to it via using `this.pluginConfig` (a prototype property available for every plugin). For example, if you pass an option `isCool: true` to the plugin, you can refer to it by using `this.pluginConfig` anywhere in your plugin's class whereby it returns an object.
- If no options are to be passed to a plugin, you must indicate that it is available so that the object notion is not broken. A YAML-compliant truthy boolean is preferable, particularly `on`. Other truthy booleans include `ON`, `yes` and `YES`, `true` and `TRUE`. For example, `packem-custom-addon-plugin: on` is correct.
- **Plugins are executed in order of definition (FIFO &mdash; First In, First Out)**. Other bundlers like webpack would pipe several loaders but Packem decided to execute plugins in order of definition to make configuring much less of a pain.

## Module Interfaces

We've dealt with event hooks and successfully used the right event to do what we needed. But what is a module interface anyway?

**A module interface is an exact in-memory copy of a mutable reference of a module from the module graph available from the LC/RC**. Note that this reference doesn't affect the filesystem i.e. it is _read-only_ and just points to a file. Just like an OS can display file metadata, Packem provides metadata for modules in the module graph as module interfaces. Not all properties can be satisfied for all files like JavaScript vs non-JavaScript file types. A module interface is always in this form:

```typescript
interface ModuleInterface {
  // The mangled ID used to refer to this module.
  id: string;
  // The absolute path of this module.
  path: string;
  // A tracker that defines whether this module is dynamic or not.
  // If it is equivalent to "root", then it is not dynamic. Otherwise, it is.
  // 
  // Quick Hack: To check if a module is dynamically imported anywhere,
  // 
  //    onModuleBundle(mod: ModuleInterface) {
  //       let isDynamicModule: boolean = mod.bundle_id !== "root";
  //       // Do something with `isDynamicModule`
  //    }
  // 
  bundle_id: string;
  // The file extension of this module.
  extension: string;
  // Content of this module which doesn't exist on non-JavaScript
  // files so don't use it if you're matching a non-text-based file type.
  content?: string;
  // An array of mangled IDs that represent a module on the flat list
  // module graph. When bundling this module with its dependencies it
  // is recursively looped over until all its dependency's sub-dependencies
  // have been exhaustively transformed then taken to the serializer (when
  // the same happens to the whole module graph).
  dependencies: string[];
}
```

## Mutations, Dangling Modules and a Flat List Module Graph

> "With great performance comes great responsibility" &mdash; Packem™.

Using a tree to define a module graph is a pain point since a single mutation would require a visitor to traverse the branches just to reach a certain module. If two or more modules require each other, things become more difficult. Controlling duplication across branches is a burden.

If watch mode is enabled in `@packem/dev-plugin`, a deletion(s) can cause module(s) not to be removed from the module graph (*dangled modules*) and vice versa. So how should things be managed? All of these manipulations from the runtime context makes it even unbearable.

This approach caused a major shift in Packem's architecture. The module graph being refactored into a flat list containing extended module interfaces is found to be easier to maintain, making issues like duplication, inter-modular/circular dependencies and code splitting a breeze. This is what a module graph as a flat list appears like during build time:

```typescript
export default [
  {
    id: "root",
    dependencies: ["sKAY2qXG"]
  },
  {
    id: "sKAY2qXG",
    bundle_parent: "root",
    dependencies: ["yFj2CkjK"]
  },
  {
    id: "yFj2CkjK",
    bundle_parent: "b34kURy3",
    dependencies: ["sKAY2qXG"]
  }
  ...
] as ModuleGraph;
```

As much as using functional patterns to update the module graph may seem _cute_, it is a huge leap into anti-performance. The general principle of Packem favors _performance over expression_ without selling away features.

## Built-in events

Here's a list of all the built-in plugin events in order of dispatch right from the beginning of the bundling cycle till the very end.

### `onStart`

**Parameters:** `ConfigurationObject`.

**Returns:** `void`.

This event is dispatched before the module graph is generated.

#### The `ConfigurationObject`

The configuration object is a slightly modified version of `packem.config.yml`. It includes a few extra fields.

```typescript
interface ConfigurationObject {
  // The root module is where the resolver starts to track
  // dependencies and rehydrate them into the module graph.
  root: string;
  // Output bundle generated by the transformer.
  output: string;
  // Absolute path to the root module.
  rootPath: string;
  // Absolute path to the output bundle.
  outputPath: string;
  // Absolute path to the output bundle's directory.
  // If the output path is `./dist/bundle.js`, the equivalent
  // `outputDir` would be the absolute path prepended to `./dist/` only.
  outputDir: string;
  // Output path file name without extension.
  // If the output directory is `./dist/bundle.main.js`, then
  // the equivalent `outputPathFileStem` would be `bundle.main`
  // i.e. without `.js`.
  outputPathFileStem: string;
}
```

```typescript
onStart(config: ConfigurationObject) {
  // Logs configuration file to stdout.
  console.log(config);
}
```

### `onGenerateModuleGraph`

**Parameters:** `ModuleGraph`.

**Returns:** `void`.

Allows manipulation of the graph before `onModuleBundle`. This method is especially useful when you wish to reset all module interfaces in a graph to fit your needs after which you could use `onModuleBundle` to get your customized version of the graph.

```typescript
onGenerateModuleGraph(graph: ModuleGraph) {
  // Clear all dependencies before you call `onModuleBundle`.
  graph.root.dependencies = [];
}
```

### `onModuleBundle`

**Parameters:** `ModuleInterface`

**Returns:** `string` Overriden content of the current module. If you need a module to export the return value prepend your return value with the string `"module.exports = "`.

One of the most commonly used events. Fires when the transformer is running through the module graph in the runtime context. It gives access to the module interface of every module in the graph.

```typescript
onModuleBundle(mod: ModuleInterface): string {
  return "module.exports = " + mod.content;
}
```

### `onBeforeTransform`

**Parameters:** `ModuleInterface`

**Returns:** `void`.

Triggered just before the transformer kicks in. If you wish to run something before a Babel preset(s) and/or plugin(s) is included use this event.

```typescript
onBeforeTransform(mod: ModuleInterface) {
  // Do something before any Babel preset(s) and/or plugin(s)
  ;
}
```

### `onAfterTransform`

**Parameters:** `transformedCode: string`. A transformed version of the input files.

**Returns:** `void`.

Triggered just after the transformer handles JavaScript file types. Useful when you want to run something after a script is transformed.

```typescript
onAfterTransform(transformedCode: string) {
  // Access to transformed code i.e. transformer's output.
  console.log(transformedCode);
}
```

### `onSuccess`

**Parameters:** None.

**Returns:** `void`.

This event is dispatched when the bundling cycle is successfully complete.

```typescript
onSuccess() {
  console.log("Successfully bundled everything! 👌");
}
```

### `onEnd`

**Parameters:** `config: ConfigurationObject`. A transformed version of `packem.config.yml` exhibiting [this format](https://github.com/packem/packem/blob/master/docs/the-plugin-system.md#the-configurationobject).

**Returns:** `void`.

After every core process is complete, this event is dispatched.

```typescript
onEnd(config: ConfigurationObject) {
  // All core process are complete. Only plugins or child
  // processes are expected to be running.
  console.log(config);
}
```

## Resources

- [Packem: Under the Hood.](https://github.com/packem/packem)
- [Packem Common Plugins.](https://github.com/packem/packem-plugins)

If you happen to fall into any issue, got lost somewhere in the docs, please report to us how we can update the docs to make it easier for readers. Alternatively, you can contribute to Packem's docs by submitting a PR or filing an issue with the `documentation` tag/filter.
