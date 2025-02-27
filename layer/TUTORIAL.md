<!-- markdownlint-disable MD041 -->
<p align="left"><img src="https://vulkan.lunarg.com/img/NewLunarGLogoBlack.png" alt="LunarG" width=263 height=113 /></p>
<p align="left">Copyright © 2015-2022 LunarG, Inc.</p>

[![Creative Commons][3]][4]

[3]: https://i.creativecommons.org/l/by-nd/4.0/88x31.png "Creative Commons License"
[4]: https://creativecommons.org/licenses/by-nd/4.0/

<p align="center"><img src="./images/logo.png" width=400 /></p>

# VK\_LAYER\_KHRONOS\_profiles

## Overview

### Vulkan capabilities test coverage with the Khronos Profiles Layer
The Khronos Profiles Layer helps test across a wide range of hardware capabilities without requiring a physical copy of every device. It can be applied without modifying any application binaries, and in a fully automated fashion. The *Profiles layer* is a Vulkan layer that can override the values returned by your application's queries of the GPU. Profiles layer uses a JSON text configuration file to make your application see a different driver/GPU than is actually in your system. This capability is useful to verify that your application both a) properly queries the limits from Vulkan, and b) obeys those limits.

The *Profiles layer* is available pre-built in the [Vulkan SDK](https://www.lunarg.com/vulkan-sdk/), and continues to evolve. It works for all Vulkan platforms (Linux, Windows, macOS, and Android). The *Profiles layer* can be enabled and configured using the [Vulkan Configurator](https://vulkan.lunarg.com/doc/sdk/latest/windows/vkconfig.html) included with the Vulkan SDK.

The role of the *Profiles layer* is to "simulate" a Vulkan implementation by modifying the features and resources of a more-capable implementation. The *Profiles layer* does not add capabilities to your existing Vulkan implementation by "emulating" additional capabilities with software; e.g. the *Profiles layer* cannot add geometry shader capability to an actual device that doesn't already provide it. Also, the Profiles layer does not "enforce" the features being simulated. You can use the Validation Layer in conjunction with the Profiles Layer to identify where your application is not adhering to the features being simulated by the Profiles Layer.

### Using the Profiles layer
The *Profiles Layer* can be controlled using environment variables or more intuitively using the *Vulkan Configurator* GUI application.

![Vulkan Configurator](https://github.com/KhronosGroup/Vulkan-Profiles/blob/master/images/vkconfig.png)
*The built-in Portability layers configurations in Vulkan Configurator using the Validation and the Profiles layers*

The *Portability* layers configuration in *Vulkan Configurator* checks that a Vulkan application follows the requirements of a *Vulkan Profile*. This layers configuration relies on the *Vulkan Profiles* layer to override the Vulkan device capabilities and the *Vulkan Validation Layer* to check that the Vulkan application doesn't rely on capabilities not supported by the selected profile.

The input to the *Profiles layer* is a profiles file that is using the flexible JSON syntax. The profiles file format is defined by a formal JSON schema, so any profiles file may be verified to be correct using freely available JSON validators. Examination of the schema file reveals the extent of parameters that are available for configuration.

To use the Profiles Layer Vulkan version 1.1 is required.

Example of a *Profiles layer* JSON profiles file: [VP_LUNARG_test_structure_simple.json](https://github.com/KhronosGroup/Vulkan-Profiles/blob/master/profiles/test/data/VP_LUNARG_test_structure_simple.json)

### Android
To enable, use a setting with the path of the profiles file to load:
```
adb shell settings put global debug.vulkan.khronos_profiles.profile_file <path/to/profiles/JSON/file>
```

Optional: use settings to enable debugging output and exit-on-error:
```
adb shell settings put global debug.vulkan.khronos_profiles.debug_reports DEBUG_REPORT_DEBUG_BIT
adb shell settings put global debug.vulkan.khronos_profiles.debug_fail_on_error true
```

## Technical Details

The *Profiles Layer* is a Vulkan layer that can modify the results of Vulkan PhysicalDevice queries based on a profiles file (JSON format), thus simulating some of the capabilities of a device by overriding the capabilities of the actual device under test.

Please note that this device simulation layer "simulates", rather than "emulates".
By that we mean that the layer cannot add emulated capabilities that do not already exist in the system's underlying actual device.
The Profiles layer will not enable a less-capable device to emulate a more-capable device.

Application code can be tested to verify it responds correctly to the capabilities reported by the simulated device.
That could include:
* Properly querying the capabilities of the device.
* Properly complying with the limits reported from the device.
* Verifying all necessary capabilities are reported present, rather than assuming they are available.
* Exercising fall-back code paths, if optional capabilities are not available.

The `fail_on_error` option can be used to make sure the device supports the requested capabilities. 
In this case if an application erroneously attempts to overcommit a resource, or use a disabled feature, the Profiles layer will return `VK_ERROR_INITIALIZATION_FAILED` from `vkEnumeratePhysicalDevices()`.

The Profiles layer will work together with other Vulkan layers, such as the Validation layer.
When configuring the order of the layers, the Profiles layer should be "last";
i.e.: closest to the driver, furthest from the application.
That allows the Validation layer to see the results of the Profiles layer, and enable Validation to flag incorrect API usage beyond the simulated capabilities.

If you find issues, please report to [Khronos' Vulkan-Profiles GitHub repository](https://github.com/KhronosGroup/Vulkan-Profiles/issues).

### Profiles Layer operation and profiles file
At application startup, during `vkEnumeratePhysicalDevices()`, the Profiles layer initializes its internal tables from the actual physical device in the system, then loads the profiles file, which specifies override values to apply to those internal tables.

JSON file formats consumed by the Profiles layer are specified by the following JSON schema https://github.com/KhronosGroup/Vulkan-Profiles/blob/master/schema/profile_schema.json

The schema permits additional top-level sections to be optionally included in profiles files;
any additional top-level sections will be ignored by the Profiles layer.

The schemas define basic range checking for common Vulkan data types, but they cannot detect whether a particular profile is illogical.
If a profile defines capabilities beyond what the actual device is natively capable of providing and the `fail_on_error` option is not used, the results are undefined.
The Profiles layer has some simple checking of profile values and writes debug messages (if enabled) for values that are incompatible with the capabilities of the actual device.

This version of the Profiles layer currently supports Vulkan v1.3 and below including all Vulkan extensions.
If the application requests an unsupported version of the Vulkan API, the Profiles layer will emit an error message.

When a Vulkan extension gets promoted so can the structures it defines. It is invalid for a profile to specify a structure and its promoted version at the same time.
In this case the Profiles layer will use the promoted version of the structure and emit a warning.

### `VK_KHR_portability_subset` Emulation

The Profiles layer provides the ability to emulate the `VK_KHR_portability_subset` extension on devices that do not implement this extension.
This feature allows users to test their application with limitations found on non-conformant Vulkan implementations.
To turn on this feature, enable it as described [below](#emulate-vk_khr_portability_subset).

### Profiles Layer Settings

#### Profiles JSON file
- Environment Variable: `VK_KHRONOS_PROFILES_PROFILE_FILE`
- `vk_layer_settings.txt` option: `khronos_profiles.profile_file`
- Android option: `debug.vulkan.khronos_profiles.profile_file`
- Default value: Not set

Name of the profiles file to load. 

#### Selecting profile
- Environment Variable: `VK_KHRONOS_PROFILES_PROFILE_NAME`
- `vk_layer_settings.txt` option: `khronos_profiles.profile_name`
- Android option: `debug.vulkan.khronos_profiles.profile_name`
- Default value: Not set

Name of the profile to load from the profiles file.

#### Profiles JSON validation
- Environment Variable: `VK_KHRONOS_PROFILES_PROFILE_VALIDATION`
- `vk_layer_settings.txt` option: `khronos_profiles.profile_validation`
- Android option: `debug.vulkan.khronos_profiles.profile_validation`
- Options: `true`, `false`
- Default value: `true`

Validate the profiles file against the Vulkan SDK profile schema if the file is found. 

#### Emulate `VK_KHR_portability_subset`
- Environment Variable: `VK_KHRONOS_PROFILES_EMULATE_PORTABILITY`
- `vk_layer_settings.txt` option: `khronos_profiles.emulate_portability`
- Android option: `debug.vulkan.khronos_profiles.emulate_portability`
- Options: `true`, `false`
- Default value: `true`

Enables emulation of the `VK_KHR_portability_subset` extension.

#### Simulating capabilities
- Environment Variable: `VK_KHRONOS_PROFILES_SIMULATE_CAPABILITIES`
- `vk_layer_settings.txt` option: `khronos_profiles.simulate_capabilities`
- Android option: `debug.vulkan.khronos_profiles.simulate_capabilities`
- Options: `SIMULATE_API_VERSION_BIT`, `SIMULATE_EXTENSIONS_BIT`, `SIMULATE_FEATURES_BIT`, `SIMULATE_PROPERTIES_BIT`, `SIMULATE_FORMATS_BIT`
- Default value: `SIMULATE_API_VERSION_BIT,SIMULATE_FEATURES_BIT,SIMULATE_PROPERTIES_BIT`

Enables modification of device capabilities.

#### Debug Actions
- Environment Variable: `VK_KHRONOS_PROFILES_DEBUG_ACTIONS`
- `vk_layer_settings.txt` option: `khronos_profiles.debug_actions`
- Android option: `debug.vulkan.khronos_profiles.debug_actions`
- Options: `DEBUG_ACTION_FILE_BIT`, `DEBUG_ACTION_STDOUT_BIT`, `DEBUG_ACTION_OUTPUT_BIT`, `DEBUG_ACTION_BREAKPOINT_BIT`
- Default value: `DEBUG_ACTION_STDOUT_BIT`

Enables different debugging actions.

#### Debug Filename
- Environment Variable: `VK_KHRONOS_PROFILES_DEBUG_FILENAME`
- `vk_layer_settings.txt` option: `khronos_profiles.debug_filename`
- Android option: `debug.vulkan.khronos_profiles.debug_filename`
- Default value: `profiles_layer_log.txt`

Sets the output location.

#### Debug file discard
- Environment Variable: `VK_KHRONOS_PROFILES_DEBUG_FILE_DISCARD`
- `vk_layer_settings.txt` option: `khronos_profiles.debug_file_discard`
- Android option: `debug.vulkan.khronos_profiles.debug_file_discard`
- Options: `true`, `false`
- Default value: `true`

Discard the content of the log file between each layer run.

#### Debug fail on error
- Environment Variable: `VK_KHRONOS_PROFILES_DEBUG_FAIL_ON_ERROR`
- `vk_layer_settings.txt` option: `khronos_profiles.debug_fail_on_error`
- Android option: `debug.vulkan.khronos_profiles.debug_fail_on_error`
- Options: `true`, `false`
- Default value: `false`

Enabled failing if an error occurs.

#### Debug reports
- Environment Variable: `VK_KHRONOS_PROFILES_DEBUG_REPORTS`
- `vk_layer_settings.txt` option: `khronos_profiles.debug_reports`
- Android option: `debug.vulkan.khronos_profiles.debug_reports`
- Options: `DEBUG_REPORT_NOTIFICATION_BIT`, `DEBUG_REPORT_WARNING_BIT`, `DEBUG_REPORT_ERROR_BIT`, `DEBUG_REPORT_DEBUG_BIT`
- Default value: 0

Enabled reports level.

#### Exclude device extensions
- Environment Variable: `VK_KHRONOS_PROFILES_EXCLUDE_DEVICE_EXTENSIONS`
- `vk_layer_settings.txt` option: `khronos_profiles.exclude_device_extensions`
- Android option: `debug.vulkan.khronos_profiles.exclude_device_extensions`
- Default value: Not set

Prevents device extensions from being reported by the Vulkan physical device.

#### Exclude formats
- Environment Variable: `VK_KHRONOS_PROFILES_EXCLUDE_FORMATS`
- `vk_layer_settings.txt` option: `khronos_profiles.exclude_formats`
- Android option: `debug.vulkan.khronos_profiles.exclude_formats`
- Default value: Not set

Prevents image formats from being reported as supported by the Vulkan physical device.

**Note:** If multiple values are set in any environment variable they are be separated by ';' on windows and with ':' on other platforms. In `layer_settings.txt` file multiple values are separated by ','.

**Note:** In desktop environments, environment variables take precedence over `vk_layer_settings.txt` options.

### Example using the Profiles layer using Linux environment variables
```bash
# Configure bash to find the Vulkan SDK.
source $VULKAN_SDK/setup-env.sh

# Set loader parameters to find and load the Profiles layer from our local Vulkan-Profiles build.
export VK_LAYER_PATH="${VulkanProfiles}/build/bin"
export VK_INSTANCE_LAYERS="VK_LAYER_KHRONOS_profiles"

# Specify the simulated device's profiles file.
export VK_KHRONOS_PROFILES_PROFILE_FILE="${VulkanProfiles}/profiles/VP_LUNARG_desktop_portability_2021.json"

# Enable notification and debug messages from the Profiles layer.
export VK_KHRONOS_PROFILES_DEBUG_REPORTS="DEBUG_REPORT_NOTIFICATION_BIT,DEBUG_REPORT_DEBUG_BIT"

# Enable capabilities to simulate
export VK_KHRONOS_PROFILES_SIMULATE_CAPABILITIES="SIMULATE_FEATURES_BIT,SIMULATE_PROPERTIES_BIT,SIMULATE_EXTENSIONS_BIT"

# Run a Vulkan application through the Profiles layer.
vulkaninfo
# Compare the results with that app running without the Profiles layer.
```
