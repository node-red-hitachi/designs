---
state: draft
---

# Lifecycle Model of Function Node

This proposal covers enhanced lifecycle model of function node.

### Authors

 - @HiroyasuNishiyama

### Details

Function node can describe algorithms in JavaScript code.  It plays a central role for expressing complex algorithms by Node-RED flows.  However, current implementation of function node has following problems:

1. Can't use external libraries without modifying settings.js,
2. Function body is executed each time message is received. 
   Describing common initialization or shutdown code is difficult.

This design note addresses these problems.

### Lifecycle Model

Following figure shows new lifecycle model of function node.  While creating a flow using function node, a user  can specify following items in settings panel:

- required npm modules

  module to be loaded and variable name for accessing loaded module object,

- initialization code
  code fragment executed at startup of the flow,

- function body code
  code fragment exected when a message is received (same one in current function node),

- finalization code
  code fragment executed when the flow is stopped.

![fn-lifecycle](./fn-lifecycle.png)

### Use of External JavaScript Libraries

New config tab includes declaration of list of JavaScript variable and required module.

![config](./config.png)

If required modules are not installed, we need a means to install NPM modules from inside Node-RED editor.  This requires considerations on security, restriction of environments, user interaction, and other topics.  In the above lifecycle model, we propose to make automatic (e.g. for headless Node-RED) or interactive module installation selectable by changing a value in `settings.js`.  

Function node declares its required NPM module list to runtime by using new API.   For interactive mode, we add NPM management UI in editor settings panel.

![require-interaction](./require-interaction.png)

Because current Node-RED implementation starts flow execution before message handler callback is ready, we may need to modify runtime execution model in order to make function nodes wait for receiving messages before required NPM modules become ready to use.

#### Additional settings for external npm module in `settings.js`

Add a top-level property named `externalModule` to `settings.js`.  It points to a object with following properties.

| name | type   | description  |
| :--- | :----- | :----------- |
| mode | string | one of `"auto"`, `"auto-update"`, `"manual"`, or `"none"` (default: `"manual"`)<br/>- `auto`: automatically install requested NPM module if not installed.<br/>- `auto-update`: same as `auto` but with automatic update if newer version is available.<br/>- `manual`: users can manually install requested NPM module from user settings panel.<br/>- `none`: installation of external NPM modules is not allowd |
| whiteList | array of string | List of regular expressions that matches module's NPM registry specification. External libraries that matches this list **can** be used in `Function` nodes. |
| blackList | array of string | List of regular expressions that matches module's NPM registry specification. External libraries that matches this list **can not** be used in `Function` nodes. |

`whiteList` precedes over `blackList`.

```
    // Example of settings.js
    externalModules: {
        mode: "manual",
        whiteList: [
            "^fs-ext$",       // allow fs-ext
            "^qrcode@1.4.4$"  // allow qrcode version 1.4.4
        ],
        blackList: [
            ".*"              // disallow others 
        ]
    },
```

#### New API for requesting NPM module

- `RED.nodes.registerRequiredModules`(*module*)

Requests Node-RED runtime to install NPM module *module*.  Returns a promise that resolves to required module object or `undefined` if installation failed.  NPM module installation is performed according to specification in `settiongs.js` described above.

If `auto` mode is specified, execution of a function node that require a NPM module waits for completion of its installation.

#### Manual installation of NPM module

User Settings panel will be extended to have `Modules` tab.
It lists up NPM modules requested by `registerRequiredModules`.  
Entry for already installed NPM module has `check update` button and `uninstall` button. Entry for not installed NPM module has `install` button. 

![modules-tab](./modules-tab.png)


### Initialization and Finalization

Add new tabs (`Initialize`/`Finalize` [tab names must be reconsidered]) for specifying code for common initialization or shouttown code.  Code executed for each received messages (same as code in current Function node) can be specified in `Function` tab.  This `Function` tab is selected by default.

![init-final](./init-fin.png)

Button for expanding code editor should be moved to bottom of settings panel in order to make editing area larger.

#### Available APIs for initialization/finalization code

The `node` variable can not be accessed from initialization/finalization code.  Therefore, it is not possible to use APIs depending on `node` such as `node.send`.  Contex APIs for `flow` or `global` context can also be used.

#### Asynchronous processing

If the initialization code needs to start an asynchronous work that needs to be resolved before the start of the function body, it is possible to return the promise from initialization code.  Initialization code is wrapped by async function, thus can use `await` withing the code.

During the initialization process, other nodes may become active and may send messages to the function node. In such a case, the accepted messages are stored until the initialization process is completed, and they are processed in the order in which they were received when the initialization process is completed.

#### Error handling 

Exceptions to the initialization code are logged to console. If error handling is necessary, it should be handled in the initialization/termination code appropriately.

### Library Enhancements

Export format of function node to Node-RED library currently uses comments to encode properties.  This must be extended to be able to include initialization/finilization code and required npm modules information.

## History

  - 2020-02-09 - Initial Note
  - 2020-04-02 - Update API access and error handling in init/final code
  - 2020-04-10 - Update async processing in initialization code
  - 2020-04-16 - Update message handling received while async initialization
  - 2020-04-20 - Update NPM installation details