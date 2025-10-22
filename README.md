# smc_verifiable_plugins

This package provides the base class and examples to develop plugins useable for Statistical Model Checking, that can be integrated into formal models for verifying them using smc_storm.

SMC plugins have to develop two operations:
* *reset*: Set the plugin to its initial state.
* *step*: Advance the automaton to its next state.

In [smc_storm](https://github.com/convince-project/smc_storm) we implemented an extension of the [JANI format](https://jani-spec.org/) to load SMC plugins and advance their state each time a specific _action-name_ from a specific _automaton_ is executed.

## Plugin development

An exemplary SMC plugin is provided [here](plugins/src/int_accumulation_smc_plugin.cpp), together with its [header class](plugins/include/int_accumulation_smc_plugin.hpp).

We refer to the [online documentation](https://convince-project.github.io/smc_verifiable_plugins/development.html) for detailed information on how to write a new SMC plugin and how to write a CMake file for compiling and exporting it.

## Integration with JANI

When associating SMC plugins to JANI, we need to define when the plugin instance needs to be updated, and how it is parametrized.

In SMC Storm, we extended the JANI format to be able to load plugins, and get their next state each time a specific automaton's action-name is executed.

SMC plugins can be configured in the `plugins` section of the JANI file. The related JSON-Schema is provided below, and shows the fields required to instantiate a single SMC plugin:

```javascript
schema(
  "plugins": Array.of(
    {
        "plugin_id": String,                    // Name of plugin to load
        "automaton_id": String,                 // Name of the automaton associated to the plugin
        "action_name": String,                  // Name of the action in the automaton that advances the plugin to the next step
        "init": Array.of(                       // Configuration (constant) parameters for initializing the plugin
        {
            "name": String,                     // Name of the init config param to configure in the plugin
            "type": ("int", "bool", "real"),
            "value": (int, bool, real)
        }),
        "input": Array.of(                      // Definition of what to provide as input to the plugin
        {
            "name": String,                      // Referring to plugin's internal names
            "type": ("int", "bool", "real"),
            "value": Expression                 // Value passed to the plugin: any valid JANI Expression works
        }),
        "output": Array.of(
        {
            "ref": String,                      // Referring to the JANI variable storing the output value
            "value": String                     // Referring to the plugin's internal name
        }),
    })
)
```

## Installation

To use this package for developing new plugins, it is enough to integrate it in your CMake project following the instructions in [the documentation](https://convince-project.github.io/smc_verifiable_plugins/development.html#fetch-this-package-in-cmake).

For active development of this package, having it built will automatically make it available to the other packages depending on it, using the `find(smc_verifiable_plugins REQUIRED)` macro from CMake.
After running that, the variable `smc_verifiable_plugins_INCLUDE_DIR` with the path to the include folder, and the variable `smc_verifiable_plugins_PLUGINS_PATH` with the path to the compiled plugins libraries folder, will be available.
Those variables can be used to locate the header files and the plugins.

The following steps are necessary to build the package:
```bash
cd smc_verifiable_plugins
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo && make -j8
```

Optionally, to ensure the build was successful, the tests can be executed:
```bash
ctest
```
