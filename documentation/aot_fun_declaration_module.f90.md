# aotus/source/aot_fun_declaration_module.f90

## Overview

The `aot_fun_declaration_module` serves a specific but crucial role in the AOTUS (Adaptive Optics Toolbox for Usage with Scripts) framework. Its primary purpose is to define the `aot_fun_type` derived data type. This type is designed to encapsulate information about functions, which could include functions defined within Lua scripts that Fortran needs to call, or Fortran functions that are exposed to the Lua environment.

The module's comment, "Helping module to define the aot_fun_type without causing dependency locks," indicates that this type definition is foundational and is likely used by multiple other modules. Placing it in a separate, minimal module helps to prevent circular dependencies that might arise if it were defined within a larger module that itself has other dependencies.

## Key Components

- **`aot_fun_type` (Derived Type):**
  This is the sole public component of the module. It is a structure designed to hold metadata about a function. Its components are:
    - **`handle (integer)`**: An integer value, likely used as a reference or handle to the function. This could be, for example, a reference to a Lua function stored in the Lua registry. It defaults to `0`.
    - **`arg_count (integer)`**: An integer storing the number of arguments that the function expects or is known to handle. It defaults to `0`.
    - **`id (integer(kind=long_k))`**: A long integer used as a unique identifier for the function. The use of `long_k` (from `flu_kinds_module`) ensures it can hold a large range of values. It defaults to `0`.

## Important Variables/Constants

The components of the `aot_fun_type` derived type are the most important elements defined by this module:
- `handle`: Stores a reference or handle.
- `arg_count`: Stores the expected number of arguments.
- `id`: Stores a unique long integer identifier.

There are no public Fortran `parameter` constants defined in this module.

## Usage Examples

The `aot_fun_type` is used to declare variables that will store information about specific functions.

```fortran
module function_manager
  use aot_fun_declaration_module
  use flu_kinds_module, only: long_k ! For literal kind if needed, though id is long_k by type
  implicit none

  type(aot_fun_type) :: my_lua_callback
  type(aot_fun_type) :: registered_fortran_routine

  subroutine setup_lua_callback(lua_func_ref, num_args, unique_id)
    integer, intent(in) :: lua_func_ref
    integer, intent(in) :: num_args
    integer(kind=long_k), intent(in) :: unique_id

    my_lua_callback%handle = lua_func_ref
    my_lua_callback%arg_count = num_args
    my_lua_callback%id = unique_id

    write(*,*) "Lua Callback Setup:"
    write(*,*) "  Handle:", my_lua_callback%handle
    write(*,*) "  Arg Count:", my_lua_callback%arg_count
    write(*,*) "  ID:", my_lua_callback%id
  end subroutine setup_lua_callback

end module function_manager

program test_fun_type
  use function_manager
  use flu_kinds_module, only: long_k ! For literal kind
  implicit none

  call setup_lua_callback(12345, 2, 9876543210_long_k)

end program test_fun_type
```

## Dependencies and Interactions

- **`flu_kinds_module`:**
    - This module is used to import the `long_k` kind parameter, which is used to define the `id` component of the `aot_fun_type`. This ensures that the `id` field can store a large integer value, suitable for unique identification.

- **Other AOTUS Modules:**
    - `aot_fun_declaration_module` is intended to be a foundational module. Other modules within the AOTUS system that deal with function registration, management, or invocation (either calling Lua functions from Fortran or exposing Fortran functions to Lua) would `use aot_fun_declaration_module` to access the `aot_fun_type` definition.
    - Its existence as a separate module helps prevent circular dependencies, which can occur if a widely used type is defined in a module that also depends on other modules that might, in turn, need the type definition.

This module plays a structural role in the AOTUS software architecture by providing a clean and non-dependency-locking definition for function metadata.
