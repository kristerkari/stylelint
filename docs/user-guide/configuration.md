# Configuration

The linter _expects a configuration object_. You can either craft your own config or extend an existing one.

## Loading Configuration

For the Node API and PostCSS plugin, you can pass a config directly or find and load it.
For the CLI, you must find and load it.

Finding and loading of your configuration object is done with [cosmiconfig](https://github.com/davidtheclark/cosmiconfig).
Starting from the current working directory, it will look for the following possible sources, in this order:

- a `stylelint` property in `package.json`
- a `.stylelintrc` file in JSON or YAML format
- a `stylelint.config.js` file exporting a JS object

Once one of these is found and parsed, the search will stop and that object will be used.

The configuration search can be short-circuited by passing a `--config` CLI argument specifying the path to your configuration file.

## The Configuration Object

The configuration object can have the following properties. Only `rules` is required.

### `rules`

[Rules](/docs/user-guide/rules.md) determine what the linter looks for and complains about.
*No rules are turned on by default*, so this is where you turn on everything you want to check.

**The `rules` property is an object whose keys are rule names and values are rule configurations.**
Each rule configuration fits one of the following formats:

- a single value (the primary option)
- an array with two values (`[primary option, secondary options]`)
- `null` (to turn the rule off)

```json
{
  "rules": {
    "color-no-invalid-hex": true,
    "declaration-colon-space-after": "always",
    "max-empty-lines": 2,
    "indentation": ["tab", {
      "except": ["value"]
    }]
  }
}
```

#### Severities: error & warning

By default, all rules have an "error"-level severity. To downgrade any rule to "warning"-level severity,
pass a secondary option `"warn": true`.

```js
// error-level severity examples
{ "indentation": 2 }
{ "indentation": [2] }

// warning-level severity examples
{ "indentation": [2, { "warn": true } ] }
{ "indentation": [2, {
    "except": ["value"],
    "warn": true
  }]
}
```

Different reporters may use these severity levels in different way, e.g. display them differently, or exit the process differently.

##### Legacy Numbered Severities

Before v3, configuration involved a system of numbered severities, like ESLint's.
Numbered severities have been deprecated (and will be removed in v4), but can still be used for the time being —
just expect a deprecation warning whenever stylelint runs.

stylelint should be able to figure out whether you're using legacy numbered severities as opposed to the
more recent standard. But there is one situation in which auto-detection won't work:
if your config *only contains rules that take integers as their primary option* (e.g. indentation).
**If this exception applies to you, you need to explicitly force legacy numbered severities by adding
`legacyNumberedSeverities: true` to your config.**

### `extends`

Your configuration can *extend* an existing configuration (whether your own or a third-party config).
When one configuration extends another, it starts with the other's properties then adds to and overrides what's there.

You can extend an array of existing configurations, with each item in the array taking precedence over the following (so the first item overrides everything else, the second item overrides everything but the first, the last gets overridden by everything else, etc.).

For example, extending the [`stylelint-config-suitcss`](https://github.com/stylelint/stylelint-config-suitcss) and then changing indentation to tabs and turning off the number-leading-zero rule:

```json
{
  "extends": "stylelint-config-suitcss",
  "rules": {
    "indentation": "tab",
    "number-leading-zero": null,
  }
}
```

Or starting with `stylelint-config-suitcss`, then extending layering `myExtendableConfig` on top of that, and then overriding the indentation rule:

```json
{
  "extends": [
    "./myExtendableConfig",
    "stylelint-config-suitcss"  
  ],
  "rules": {
    "indentation": "tab",
  }
}
```

**The value of `"extends"` is a "locater" (or an array of "locaters") that is ultimately `require()`d, so can fit whatever format works with Node's `require.resolve()` algorithm.** That means the a "locater" can be:

- The name of a module in `node_modules` (e.g. `stylelint-config-wordpress`; that module's `main` file must be a valid JSON configuration)
- An absolute path to a file (which makes sense if you're creating a JS object in a Node context and passing it in) with a `.js` or `.json` extension.
- A relative path to a file with a `.js` or `.json` extension, relative to the referencing configuration (e.g. if configA has `extends: "../configB"`, we'll look for `configB` relative to configA).

*Because of `extends`, you can create and use shareable stylelint configurations.*

### `plugins`

[Plugins](/docs/user-guide/plugins.md) are userland rules that support _non-standard_ CSS features, or very specific use cases. To use one, add a `"plugins"` array to your config, containing "locaters" identifying the plugins you want to use.
As with `extends`, above, a "locater" can be either an npm module name, an absolute path, or a path relative to the invoking configuration file.

Once the plugin is declared, within your `"rules"` object you can add settings for the plugin's rule just like any standard rule. You will have to look at the plugin's documentation to know what the rule name should be.

```json
{
  "plugins": [
    "../special-rule.js",
  ],
  "rules": {
    "special-rule": "everything",
  },
}
```

### `ignoreFiles`

Provide a glob or array of globs to ignore specific files.

These globs are analyzed relative to

- the config's filepath, if the config is a file,
- or `process.cwd()`

The `ignoreFiles` property is stripped from extended configs: only the root-level config can ignore files.

## Configuring rules

[Rules](/docs/user-guide/rules.md) are built into the linter and focus on _standard_ CSS. They are configured within the `rules` key of the config.

### Turning rules on and off

Each rule can be turned off or on:

* `null` - turn the rule off.
* `true|options` - turn the rule on.

All turned-on rules error by default. You can reduce the severity of a rule, to a warning, by adding `"warn": true` to its secondary options.

An example of turning rules on and off:

```js
{
  "rules": {
    "rule-no-single-line": null, // turn rule off
    "declaration-no-important": true, // turn rule on
    "block-no-empty": [ true, { "warn": true } ], // turn rule on as warning
    "indentation": "tabs" // turn on a rule that has options
  }
}
```

Rules can be temporarily turned off by using special comments in your CSS. For example, you can either turn all the rules off:

```css
/* stylelint-disable */
a {}
/* stylelint-enable */
```

Or you can turn off individual rules:

```css
/* stylelint-disable selector-no-id, declaration-no-important  */
#id {
  color: pink !important;
}
/* stylelint-enable */
```

### Configuring options

Only the `*-no-*` rules don't expect options. All the other rules must be explicitly configured as _there are no default values_.

An example of explicitly configuring the options for three rules:

```js
{
  "rules": {
    "indentation": "tab", {
      "except": ["value"],
    }],
    "declaration-colon-space-before": "never",
    "number-leading-zero": "always",
  }
}
```

Remember that any rule can be downgraded to "warning"-level severity by adding `"warn": true` as a secondary option.

### Rules work together

The `*-before` and `*-after` whitespace rules can be used together to enforce strict conventions.

Say you want to enforce no space before and a single space after the colon in every declaration:

```css
    a { color: pink; }
/**          ↑
 * No space before and a single space after this colon */
```

You can enforce that with:

```js
"declaration-colon-space-after": "always",
"declaration-colon-space-before": "never",
```

Some *things* (e.g. declaration blocks and value lists) can span more than one line. In these cases `newline` rules and extra options can be used to provide flexibility.

For example, this is the complete set of `value-list-comma-*` rules and their options:

* `value-list-comma-space-after`: `"always"|"never"|"always-single-line"|"never-single-line"`
* `value-list-comma-space-before`: `"always"|"never"|"always-single-line"|"never-single-line"`
* `value-list-comma-newline-after`: `"always"|"always-multi-line|"never-multi-line"`
* `value-list-comma-newline-before`: `"always"|"always-multi-line"|"never-multi-line"`

Where `*-multi-line` and `*-single-line` are in reference to the value list (the *thing*). For example, given:

```css
a,
b {
  color: red;
  font-family: sans, serif, monospace; /* single line value list */
}              ↑                    ↑
/**            ↑                    ↑
 *  The value list start here and ends here */
```

There is only a single-line value list in this example. The selector is multi-line, as is the declaration block and, as such, also the rule. But the value list isn't and that is what the `*-multi-line` and `*-single-line` refer to in the context of this rule.

#### Example A

Say you only want to allow single-line value lists. And you want to enforce no space before and a single space after the commas:

```css
a {
  font-family: sans, serif, monospace;
  box-shadow: 1px 1px 1px red, 2px 2px 1px 1px blue inset, 2px 2px 1px 2px blue inset;
}
```

You can enforce that with:

```js
"value-list-comma-space-after": "always",
"value-list-comma-space-before": "never",
```

#### Example B

Say you want to allow both single-line and multi-line value lists. You want there to be a single space after the commas in the single-line lists and no space before the commas in both the single-line and multi-line lists:

```css
a {
  font-family: sans, serif, monospace; /* single-line value list with space after, but no space before */
  box-shadow: 1px 1px 1px red, /* multi-line value list ... */
    2px 2px 1px 1px blue inset, /* ... with newline after, ...  */
    2px 2px 1px 2px blue inset; /* ... but no space before */
}
```

You can enforce that with:

```js
"value-list-comma-newline-after": "always-multi-line",
"value-list-comma-space-after": "always-single-line",
"value-list-comma-space-before": "never",
```

#### Example C

Say you want to allow both single-line and multi-line value lists. You want there to be no space before the commas in the single-line lists and always a space after the commas in both lists:

```css
a {
  font-family: sans, serif, monospace;
  box-shadow: 1px 1px 1px red
    , 2px 2px 1px 1px blue inset
    , 2px 2px 1px 2px blue inset;
}
```

You can enforce that with:

```js
"value-list-comma-newline-before": "always-multi-line",
"value-list-comma-space-after": "always",
"value-list-comma-space-before": "never-single-line",
```

#### Example D

Lastly, the rules are flexible enough to enforce entirely different conventions for single-line and multi-line lists. Say you want to allow both single-line and multi-line value lists. You want the single-line lists to have a single space before and after the colons. Whereas you want the multi-line lists to have a single newline before the commas, but no space after:


```css
a {
  font-family: sans , serif , monospace; /* single-line list with a single space before and after the comma */
  box-shadow: 1px 1px 1px red /* multi-line list ... */
    ,2px 2px 1px 1px blue inset /* ... with newline before, ...  */
    ,2px 2px 1px 2px blue inset; /* ... but no space after the comma */
}
```

You can enforce that with:


```js
"value-list-comma-newline-after": "never-multi-line",
"value-list-comma-newline-before": "always-multi-line",
"value-list-comma-space-after": "always-single-line",
"value-list-comma-space-before": "always-single-line",
```
