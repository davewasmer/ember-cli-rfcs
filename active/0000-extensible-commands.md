- Start Date: 2015-3-4
- RFC PR:
- ember-cli Issue:

# Summary

We need the ability to extend the built-in base commands supplied by Ember CLI (i.e. `serve`, `build`, etc) through addons.

# Motivation

There are two use cases that sparked this:

1. `mocha` provides a `--grep/-g` option to run tests that match a supplied regex. This is different behavior from the standard `--filter` option allowed by `ember test`. Right now, the `--grep` feature is essentially unusable without custom test commands (i.e. `ember mocha:test`) or hacking the `--filter` option (i.e. `--filter="^"` indicates a regex). Ideally, `ember-cli-mocha` would be able to extend the built-in `test` command to add a `--grep` option.
2. An addon with a post-build hook that wants to modify the build output (e.g. feature flags) might want to accept additional options.

In generally, these use cases are examples of the same idea: the Ember CLI built-in commands perform common tasks that might need different implementations across projects.

# Detailed design

Addons can currently specify commands to extend the CLI interface via the `includedCommands` hook. When a command is invoked, the `lib/cli/lookup-command.js` searches through all the loaded addon commands, as well as built-in commands, for a match.

This lookup process would be refactored to use the DAG structure of loaded addons to build a flattened list of addons that implement the requested command. This stack would then be recursively invoked, such that each addon's version of the command can call `this._super()` to pass control to the next addon in the "stack". The bottom of the stack would be the built-in implementation (if one exists).

This structure mimics Connect-style middleware, where each function in the stack can decide to pass control further, or halt the recursion by not calling `this._super()`.

Here's a rough sketch in code:

```
var command, commandOptions;
var orderedAddonList = project.addons;
var commandStack = orderedAddonList.filter(function(addon) {
        return addon.hasCommand(command);
    }).map(function(addon) {
        return addon.getCommand(command);
    });

var context = { _super: invokeNextCommandInStack };
function invokeNextCommandInStack() {
    var nextCommand = commandStack.shift();
    return nextCommand.call(context, command, commandOptions);
}

invokeNextCommandInStack()
```

> **Note:** obviously, there's a lot more that should happen, and the commands are not functions themselves. Likely, either the Command model would need to incorporate the idea of a "stack", the `/lib/cli/cli.js` file would have to become stack-aware, or we introduce a new CommandStack abstraction to handle this.

# Drawbacks

* Extending built-in commands could result in user confusion, i.e. I checkout an Ember CLI project and run `ember test` and it performs unexpected behaviors, so I Google around for "ember test" and none of the standard docs help me.
* Extending built-in commands could be a bit of a footgun - it would allow the addon developer to deviate from the conventions that this project attempts to cultivate.
* Using the implementation approach outlined above, the addon that fires first for a built-in command could forgot to call super, causing a break in the chain and the default functionality not to run.

# Alternatives

* Rather than allowing complete overrides of commands, we could just provide hooks for common operations (i.e. modify the `availableOptions` hash, `anonymousOptions` array, etc). This would help prevent abuse, but limits use cases.
* Even more restrictive, we could not allow any addon modification, and instead wait for common patterns or use cases to arise, and bake those into the built-in commands via optional arguments, auto-detecting addons, config, etc. This is difficult over the long term though, and doesn't scale well.
* Finally, we may not want to allow customization of built-in commands at all, if the potential for abuse and surprising behavior outweighs the benefits of customization.

# Unresolved questions

* Is the addon load order *necessarily* the same as the addon command "inheritance" order? Is there ever a case where Addon A wants to load *after* Addon B, but appear *before* Addon B in the command inheritance stack?