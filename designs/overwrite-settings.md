---
state: draft
---

# Overwrite Settings

## Summary

In some cases, users want to change default values in `settings.js` file without modifying the file contents.  This design note proposed new command line option for changing values in `settings.js`.

## Authors

 - @HiroyasuNishiyama

## Details

Add startup option to overwrite values in `settings.js`.

`-D `*\<property path\>*`=`*\<JSON-value\>*

`-D @`*\<path-to-JSON-file\>*

First option type overwrites specified property in the `settings.js` by the specified JSON value.

Second option type overwrites properties in the `settings.js` by contents of the specified JSON file.   Objects in the `settings.js` and the JSON file are merged. So, other properties are untouched.  

The `settings.js` file is not modified with these options.

`--define` can be used instead of `-D`.

If there is an error in processing option, Node-RED stops execution.

### Examples

```
// enable project feature
-D editor.theme.projects.enabled=true

// set console log level to "debug"
-D logging.console.level="debug"

// enable context storage
// [context-setting.json]
// "contextStorage": {
//     "default": {
//         "module": "localfilesystem"
//     },
// }
-D @context-setting.json
```

### Restrictions

Values that can be specified are limited to JSON value.  Thus, settings that require JavaScript code execution (e.g. module require) can not be expressed.

## History

  - 2020-02-10 - Initial Design Note
  - 2020-03-07 - Update
