# aotus/LuaFortran/lua_parameters.f90

## Overview

The `lua_parameters` module provides a set of Fortran `parameter` constants that mirror definitions found in the Lua C header files (`lua.h` and `luaconf.h`). These parameters are crucial for the Fortran-to-Lua interface, ensuring type compatibility for numerical data and providing access to special Lua values and type identifiers from within Fortran.

The module highlights that some parameters, particularly `lua_int` and `lua_num`, are system-dependent and might require manual adaptation. Furthermore, all constants should be verified against the specific Lua library version being used to maintain consistency.

## Key Components

This module does not define any Fortran procedures or derived types. Its sole purpose is to declare public `parameter` constants for use by other modules in the Fortran-Lua interface.

## Important Variables/Constants

- **System-Dependent Kind Parameters:**
    - **`lua_int = c_long`**: Defines the Fortran kind parameter for integers that are intended to match Lua's default integer type (`lua_Integer` in C). It defaults to `c_long` from `iso_c_binding`.
    - **`lua_num = c_double`**: Defines the Fortran kind parameter for real numbers that are intended to match Lua's default number type (`lua_Number` in C). It defaults to `c_double` from `iso_c_binding`.
    *These may need to be adjusted based on the specific system architecture and Lua compilation options.*

- **Lua Configuration Constant (from `luaconf.h`):**
    - **`LUAI_MAXSTACK = 1000000`**: An integer constant representing the maximum Lua stack size. Used in the calculation of `LUA_REGISTRYINDEX`.
    *This value should match the `LUAI_MAXSTACK` definition in the `luaconf.h` of the linked Lua library.*

- **Lua Type Constants (from `lua.h`):**
    These integer constants represent the types of values in Lua:
    - `LUA_TNONE          = -1`
    - `LUA_TNIL           =  0`
    - `LUA_TBOOLEAN       =  1`
    - `LUA_TLIGHTUSERDATA =  2`
    - `LUA_TNUMBER        =  3`
    - `LUA_TSTRING        =  4`
    - `LUA_TTABLE         =  5`
    - `LUA_TFUNCTION      =  6`
    - `LUA_TUSERDATA      =  7`
    - `LUA_TTHREAD        =  8`
    *These values must align with the `LUA_T*` macros in `lua.h`.*

- **Lua Special Index Constant (from `lua.h`):**
    - **`LUA_REGISTRYINDEX  = -LUAI_MAXSTACK - 1000`**: An integer constant representing the pseudo-index for accessing the Lua registry. The registry is a pre-defined table that can be used by C (and thus Fortran) code to store Lua values that need to be referenced across C/Fortran calls.
    *Its definition depends on `LUAI_MAXSTACK`.*

## Usage Examples

The parameters defined in this module are not typically used directly in application code but are essential for the lower-level Fortran-Lua interface modules like `lua_fif.f90` and the higher-level wrappers in `flu_binding.f90`.

```fortran
! Conceptual usage within another module (e.g., flu_binding or lua_fif)
module example_lua_interaction
  use lua_parameters
  use, intrinsic :: iso_c_binding
  implicit none

  ! Declaring variables with Lua-compatible kinds
  integer(kind=lua_int) :: my_lua_integer
  real(kind=lua_num)    :: my_lua_number

  ! Checking a type returned from a Lua call (value_type would come from lua_type)
  integer :: value_type
  ! ... (value_type is obtained from a call like lua_type(L, index)) ...
  if (value_type == LUA_TSTRING) then
    ! Process a string
  else if (value_type == LUA_TNUMBER) then
    ! Process a number
    my_lua_number = 1.23_lua_num ! Assigning a literal with the specific kind
  end if

  ! Referencing the Lua registry (L_ptr is a c_ptr to lua_State)
  ! call lua_rawgeti(L_ptr, LUA_REGISTRYINDEX, some_reference)

end module example_lua_interaction
```

## Dependencies and Interactions

- **`iso_c_binding` (Intrinsic Module):** Used to access standard C kind parameters (`c_long`, `c_double`, `c_int`) which form the basis for `lua_int`, `lua_num`, and the Lua constants.

- **Lua C Header Files (`lua.h`, `luaconf.h`):** The definitions in this Fortran module are direct translations of macros and definitions from these C header files. The accuracy of these Fortran parameters is critical and directly tied to the version of the Lua library being used. The module's comments explicitly warn that these values might need to be updated if the Lua library version changes or for different system configurations.

- **`lua_fif.f90`:** This module, providing direct Fortran interfaces to Lua C functions, depends on `lua_parameters` for declaring arguments and return values with the correct kinds (e.g., `lua_pushinteger` takes an `integer(kind=lua_int)`).

- **`flu_binding.f90`:** The higher-level Fortran wrapper module also relies on `lua_parameters` for defining its interface variables and for interpreting type codes returned by Lua (e.g., `flu_type` returns an integer that is compared against `LUA_T*` constants).

This module acts as a centralized point for Lua-specific constants and type kinds, promoting consistency and maintainability within the Fortran-Lua binding ecosystem. However, its strong dependency on external C header files means it's a part of the interface that is particularly sensitive to changes in the Lua library version.
