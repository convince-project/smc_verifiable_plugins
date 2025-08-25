Develop your plugins
====================

Import this package in your project
-----------------------------------

In case you want to develop your own plugins, or add support for plugins in your model checker, you will need to make a new plugin class inheriting from the base class `SmcPluginBase`, available `here <https://github.com/convince-project/smc_verifiable_plugins/blob/main/include/smc_verifiable_plugins/smc_plugin_base.hpp>_`.

This can be achieved either by automatically fetching this package in your CMake project (recommended) or by building this package on your machine.

Fetch this package in CMake
~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the CMakeLists.txt file related to your project, you can add the following snippet to automatically download this package and make it available:

.. code-block:: cmake

    FetchContent_Declare(
        smc_verifiable_plugins
        GIT_REPOSITORY https://github.com/convince-project/smc_verifiable_plugins
        GIT_TAG 0.0.1
    )
    FetchContent_MakeAvailable(smc_verifiable_plugins)

At this point, for each executable and library that needs to use this library, you can add the dependency to this package:

.. code-block:: cmake

    add_library(<your_plugin_name> SHARED <related-src-files>)
    target_link_libraries(<your_plugin_name> PUBLIC smc_verifiable_plugins <other-dependencies>)

Installing the package locally
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Alternatively, if you want to install the package locally, you have to first build this package:

.. code-block:: bash

    cd <path-to-this-repo>
    mkdir build
    cd build
    cmake .. -DCMAKE_BUILD_TYPE=Release && make -j4

And, in the project depending on this library, do the following:

.. code-block:: cmake

    find_package(smc_verifiable_plugins REQUIRED)

    add_library(<your_plugin_name> SHARED <related-src-files>)
    target_link_libraries(<your_plugin_name> PUBLIC smc_verifiable_plugins <other-dependencies>)

Develop your plugin
--------------------

Let's look at how to develop a simple SMC Plugin by looking at the `exemplary plugin <https://github.com/convince-project/smc_verifiable_plugins/blob/main/plugins/include/int_accumulation_smc_plugin.hpp>_` `IntAccumulationSmcPlugin` provided in this package.

.. code-block:: cpp

    #pragma once
    #include "smc_verifiable_plugins/smc_plugin_base.hpp"
    #include <vector>

    namespace smc_verifiable_plugins {

    class IntAccumulationSmcPlugin : public SmcPluginBase {
    public:
        IntAccumulationSmcPlugin();

        ~IntAccumulationSmcPlugin();

        std::string getPluginName() const override {
            return "int_accumulation_smc_plugin";
        }

    private:
        void processInitParameters(const DataExchange& config) override;

        std::optional<DataExchange> processReset() override;

        std::optional<DataExchange> processInputParameters(const DataExchange& input_data) override;

        int64_t _current_state = 0U;
        const std::string _input_label;
        const std::string _output_label;
    };
    }  // namespace smc_verifiable_plugins

The first thing to notice is that we included the base class `SmcPluginBase` header file, and we inherited the new plugin from there.

Additionally, we declared the constructor, destructor and, more importantly, the `getPluginName`, `processInitParameters`, `processReset` and `processInputParameters` methods, that are overriding the ones from the base class.

In general, these methods are everything there is to be developed when making a new plugin.

The DataExchange type
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This type is the communication interface between a model checker and a plugin.

It consists of a `std::unordered_map<std::string, std::variant<int64_t, double, bool>>`, that serves as a map between a `label` and a `value`, that can be either an integer, a real or a boolean value.

The return value of plugins' operations (i.e. the ones from the `processReset` and `processInputParameters` methods) are optional, and in case of nullopt are interpreted as an error from the plugin side. In this case, the model checker is expected to consider the trace result as "Unknown" and start a new one.

The methods to override
~~~~~~~~~~~~~~~~~~~~~~~

getPluginName
______________

This function returns a string, telling the name of the plugin. This needs to match with the library name defined in CMake.

processInitParameters
_____________________

This method is in charge of configuring the specific instance of the plugin.

In this example, we expect that no configuration parameter is defined, so our method will look like this:

.. code-block:: cpp

    void IntAccumulationSmcPlugin::processInitParameters(const DataExchange& config) {
        if (!config.empty()) {
            throw std::invalid_argument("Invalid configuration provided. Expected no params.");
        }
    };

but one could, for example, configure a max. bound for the accumulated value, or add a configuration to always consider the absolute value, or more sophisticated things, depending on the application.

processReset
______________

This method is executed each time we want to start a new simulation (or trace) in the model checker, and therefore we need to reset the model to its initial state. This must affect also the plugin's output variables, therefore we need to return a `DataExchange` variable.

For our example, we want the accumulated value to be reset to its initial value, that is 0, and then report this as a return value:

.. code-block:: cpp

    std::optional<DataExchange> IntAccumulationSmcPlugin::processReset() {
        _current_state = 0U;
        return std::make_optional<DataExchange>({{_output_label, _current_state}});
    }

processInputParameters
_______________________

The last method we need to implement, is the one processing an input and providing the resulting output.

The amount of output variables provided by this method and the `processReset` one should match.

In this example, all we want is to read the input value, sum it to the internal accumulated value and return that sum in the output variable.

This is achieved with the following snippet:

.. code-block:: cpp

    std::optional<DataExchange> IntAccumulationSmcPlugin::processInputParameters(const DataExchange& input_data) {
        if (input_data.size() != 1U && !input_data.contains(_input_label)) {
            throw std::invalid_argument("Invalid input provided. Expected only one integer.");
        }
        _current_state += std::get<int64_t>(input_data.at(_input_label));
        return std::make_optional<DataExchange>({{_output_label, _current_state}});
    }

Generate the plugins's loader
_____________________________

The last thing to do, is to call the GENERATE_PLUGIN_LOADER macro, to generate the function used to load the plugin at runtime.

In this case, it looks like this:

.. code-block:: cpp

    GENERATE_PLUGIN_LOADER(IntAccumulationSmcPlugin);

The definition of this macro can be found in the `utils.hpp file <https://github.com/convince-project/smc_verifiable_plugins/blob/main/include/smc_verifiable_plugins/utils.hpp>`.
