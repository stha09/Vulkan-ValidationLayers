# Shader Validation

Shader validation is run at `vkCreateShaderModule` and `vkCreate*Pipelines` time. It makes sure both the `SPIR-V` is valid
as well as the `VkPipeline` object interface with the shader. Note, this is all done on the CPU and different than
[GPU-Assisted Validation](gpu_validation.md).

## Standalone VUs with spirv-val

There are many [VUID labeled](https://registry.khronos.org/vulkan/specs/1.3-extensions/html/vkspec.html#spirvenv-module-validation-standalone) as `VUID-StandaloneSpirv-*` and all the
[Built-In Variables](https://registry.khronos.org/vulkan/specs/1.3-extensions/html/vkspec.html#interfaces-builtin-variables)
VUIDs that can be validated on a single shader module and require no runtime information.

All of these validations are passed off to `spirv-val` in [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools/).

## spirv-opt

There are a few special places where `spirv-opt` is run to reduce recreating work already done in `SPIRV-Tools`.

## Different sections

The code is currently split up into the following main sections

- `layers/shader_instruction.cpp`
    - This contains information about each SPIR-V instruction.
- `layers/shader_module.cpp`
    - This contains information about the `VkShaderModule` object
- `layers/shader_validation.cpp`
    - This takes the following above and does the actual validation. All errors are produced here.
    - `layers/generated/spirv_validation_helper.cpp`
        - This is generated file provides a way to generate checks for things found in the `vk.xml` related to SPIR-V
- `layers/generated/spirv_grammar_helper.cpp`
    - This is a general util file that is [generated](generated_code.md) from the SPIR-V grammar

### Design details

When dealing with shader validation there are a few concepts to understand and not confuse

- `EntryPoints`
  - Tied to a shader stage (fragment, vertex, etc)
  - Knows which variables and instructions are touched in stage
    - There might be things in a `ShaderModule` not related to shader stage validation
- `Shader Module`
  - a `VkShaderModule` object
  - contains all information about the SPIR-V module
  - contains SPIR-V instructions (in an array of `uint32_t` words)
  - knows the relationship between instructions
  - can contain multiple `EntryPoints`
    - [For more details](https://github.com/KhronosGroup/SPIRV-Guide/blob/master/chapters/entry_execution.md#instructions-with-multiple-execution-modes)
- `Pipeline`
  - contains 1 or more shader object
  - decides both which `Shader Module` and `EntryPoint` are used
  - has other state not known if validating just the shader object
- `Pipeline Library` (GPL) (`VK_EXT_graphics_pipeline_library`)
  - part of a pipeline that can be reused
- `ShaderModuleIdentifier` (`VK_EXT_shader_module_identifier`)
  - lets app use a hash instead of having the driver re-create the `ShaderModule`
  - not possible to validate as the VVL don't know what the `ShaderModule` is

When dealing with validation, it is important to know what should be validated, and when.

If validation only cares about... :

- the SPIR-V itself, is mapped to the `Shader Module`
- if two stages interface, needs to be done when all stages are there
  -For `Pipeline Library` it might need to wait until linking
- descriptors variables, use `EntryPoint`
- the stage of a shader module is always known, regardless of even using `ShaderModuleIdentifier`