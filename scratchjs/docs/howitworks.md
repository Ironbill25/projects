# How ScratchJS Works
ScratchJS is an extension that works on vanilla Scratch. It adds many blocks and features to Scratch, making it way more powerful.  
However, many people that want to make their own extensions won't understand the internal workings of ScratchJS, so this explanation will make it clearer.

## 1. The VM
Scratch's VM is not exposed globally by default, so we have to use our own logic to get it.  
ScratchJS does this by climbing the React fiber tree to find the VM instance.  
The code below shows how this works.
```javascript
function waitForVM(callback) {
      if (window.vm) {
        callback(window.vm);
        // If the VM is already available, use it
      }
      vmtries++;
      if (vmtries > 15) {
        // Give up
        return;
      }
      console.log("waiting for VM, try " + vmtries);
      const el = document.querySelector(
        'div[class*="stage-header_stage-header-wrapper"]',
      ); // This is where we start the search for the VM.
      if (!el) return console.log("No stage header found"); // If the stage header is not found, we can't continue.

      const reactKey = Object.keys(el).find(
        (k) =>
          k.startsWith("__reactFiber$") ||
          k.startsWith("__reactInternalInstance$"),
      ); // Find the React key
      console.log("Check 1 - reactKey:", reactKey);
      if (!reactKey) return console.log("No react key found"); // If there is no React key, we also cannot continue

      let fiber = el[reactKey]; 
      console.log("Check 2 - fiber:", fiber, fiber.memoizedProps); // Log the fiber and its props
      while (fiber && (chkKey(fiber.memoizedProps, "ariaLabel") !== "Stage")) {
        fiber = fiber.return; // This is the main loop that climbs the tree and finds the VM.
        if (fiber?.stateNode?.props?.vm) break; // If we find the VM, break the loop.
      }
      console.log("Check 3 - fiber after loop:", fiber);
      let vm =
        fiber?.stateNode?.props?.vm ||
        fiber?.return?.return?.return?.return?.updateQueue?.stores?.[0]?.value
          ?.vm; // Massive fallback from legacy code that might still work. However we still don't know. 
      console.log("Check 4 - vm:", vm);

      if (!vm && fiber?.memoizedProps) {
        vm = fiber.memoizedProps.vm;
        console.log("Check 5 - vm from memoizedProps:", vm);
      }

      if (vm) { // If we've found the VM, expose it globally for other extensions
        window.vm = vm;
        callback(vm); // Call the callback with the VM
      } else {
        setTimeout(() => waitForVM(callback), 1000); // If we haven't found the VM, try again in 1 second
      }
    }
```

## 2. The Functions

### 2.1. The Block function
The Block function is used to define blocks and their properties, like the opcode, name, arguments, and function.
The function takes these arguments:
- blockType: The type of the block. Use the global BlockType enum.
  Example: `BlockType.COMMAND`
- opcode: The opcode of the block (like the ID)  
  Example: `myblock`
- name: The name of the block, with arguments in brackets.  
  Example: `My block [argument] more text... [anotherargument]`
- args: An object containing the arguments of the block.  
  Example: `{argument: Argument("string", "default value"), anotherargument: Argument("number", 3.14)}`
- fun: The function to run when the block is run. The arguments are passed as an object.
  Example: `function({argument, anotherargument}) { console.log(argument, anotherargument); }`

### 2.2. The Argument function
The Argument function is used to define arguments with a type and a default value.
The function takes these arguments:
- type: The type of the argument. Use the global ArgumentType enum (or a string if you already know them).
  Example: `ArgumentType.STRING`
- defaultValue: The default value of the argument.
  Example: `"default value"`

### 2.3. The Menu function
The Menu function is used to define menus for dropdown arguments.  
Menus are not defined in the block array, instead they are defined in the `menu` object of the `getInfo` function.
The function takes these arguments:
- items: An array of options for the menu.
  Example: `[MenuItem("Option 1", "option1"), MenuItem("Option 2", "option2"), MenuItem("Option 3", "option3")]`
- defaultValue: The default value of the menu.  
  Example: `"option1"`

### 2.4. The MenuItem function
The MenuItem function is used to define menu items for dropdown arguments.
The function takes these arguments:
- text: The text of the menu item, displayed to the user.
  Example: `"Option 1"`
- value: The value of the menu item, similar to how a block has an opcode.
  Example: `"option1"`

### 2.5. The ArgumentWithMenu function
This function adds an argument with a menu to a block.
The function takes these arguments:
- type: The type of the argument. Use the global ArgumentType enum (or a string if you already know them).
  Example: `ArgumentType.STRING`
- defaultValue: The default value of the argument.
  Example: `"default value"`
- menu: The menu to use for the argument, defined in the `getInfo` function's `menu` object.
  Example: `Menu([MenuItem("Option 1", "option1"), MenuItem("Option 2", "option2"), MenuItem("Option 3", "option3")], "option1")`
- acceptReporters: Whether or not the dropdown lets you place reporter/boolean blocks in it.  
  It is recommended to enable this for flexibility. The default value is `true`.
  Example: `true`

## 3. The Enums

### 3.1. The BlockType enum
The BlockType enum is used to define the type of a block.
The enum has these values:
- COMMAND: A block that runs when clicked.
- REPORTER: A block that returns a value.
- BOOLEAN: A block that returns a boolean value. Technically these can return any value, but it is not recommended.
- HAT: A block that runs when a specific event occurs.
- EVENT: A block that triggers an event.
- CONDITIONAL: A block that runs when a condition is met.
- LOOP: A block that runs multiple times.
- BUTTON: Instead of a block, places a button in the palette.  
  Note that buttons currently **DO NOT** work in vanilla Scratch, so we default these to reporter blocks instead.  
  However, most mods like TurboWarp and its mods support them.

### 3.2. The ArgumentType enum
The ArgumentType enum is used to define the type of an argument.
The enum has these values:
- STRING: A string argument. Text can be entered in here, or a reporter/boolean block can be placed inside.
- NUMBER: A number argument. Only numbers can be entered in here. A reporter/boolean block can also be placed inside.
- BOOLEAN: A boolean argument. Only boolean blocks can be placed inside. Reporters do not work inside.
All of these accept reporters/boolean blocks by default:
- COLOR: A color argument. A color picker can be opened to select a color.
- MATRIX: A matrix argument. A 5x5 matrix editor can be opened to edit the matrix.
- NOTE: A note argument. A piano note picker can be opened to select a note.
- ANGLE: An angle argument. Scratch's angle picker can be opened to select an angle.

### 3.3. The ReporterScope enum
The ReporterScope enum is used to define the scope of a reporter block.
The enum has these values:
- GLOBAL: A reporter block that can be used anywhere, including the stage.
- TARGET: A reporter block that is specific to the target. These usually do not work on the stage.

### 3.4. The TargetType enum
This enum has only two values:
- SPRITE: A sprite target.
- STAGE: The stage target.

## 4. The getInfo function
The getInfo function is used to define the extension's data. This is placed in the extension class.  
This function normally is supposed to return an object with the data.  
The data looks like this:
```javascript
id: "myextension", /* The ID of the extension. This is used to identify the extension internally.
In vanilla Scratch, the only IDs we know work is "math".
There are probably many more that work, but we don't know them. */
name: "My Extension", /* The name of the extension. This is displayed in the extension list. */
color1: "#4C9700", /* The primary color of the extension. This is used for the extension's category. */
color2: "#387D00", /* The secondary color of the extension. */
color3: "#2E6D00", /* The tertiary color of the extension. */
docsURI: "https://example.com/docs", /* The URI of the extension's documentation. This doesn't seem to do anything in vanilla Scratch but most mods add a button to visit this URI. */
blocks: [
    /* The blocks of the extension. */
    Block("myblock", "My Block [arg1]", "REPORTER", {
        arg1: {
            type: ArgumentType.STRING,
            defaultValue: "Hello"
        }
    })
],
menus: {
    /* The menus of the extension. */
    mymenu: Menu([MenuItem("Option 1", "option1"), MenuItem("Option 2", "option2")], "option1")
}
```