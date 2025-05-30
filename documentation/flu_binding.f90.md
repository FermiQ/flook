# aotus/LuaFortran/flu_binding.f90

## Overview

This module provides a comprehensive Fortran interface to the Lua C API. It defines a `flu_State` derived type to encapsulate the Lua state (`lua_State*`) and make it accessible from Fortran. The module offers a wide range of wrapper functions, generally following the Lua C API naming convention but replacing the `lua_` prefix with `flu_`. This allows Fortran programs to create, manage, and interact with Lua environments, execute Lua scripts, call Lua functions, and exchange data.

The documentation for the underlying C functions can typically be found by replacing the `flu_` prefix with `lua_` and referring to the official Lua API documentation.

## Key Components

- **`flu_State` (Derived Type):**
    - Encapsulates the Lua state (`type(c_ptr) :: state`).
    - Tracks whether standard libraries have been opened (`logical :: opened_libs`).
    - This is the primary handle used by most functions in this module to interact with a specific Lua instance.

- **`cbuf_type` (Derived Type):**
    - Holds a C pointer (`type(c_ptr) :: ptr`) and a Fortran character pointer (`character, pointer :: buffer(:)`) to manage data buffers returned from C, particularly used by `flu_dump_toBuf`.

- **Lua Type Constants (Public Parameters):**
    - `FLU_TNONE`, `FLU_TNIL`, `FLU_TBOOLEAN`, `FLU_TLIGHTUSERDATA`, `FLU_TNUMBER`, `FLU_TSTRING`, `FLU_TTABLE`, `FLU_TFUNCTION`, `FLU_TUSERDATA`, `FLU_TTHREAD`. These constants allow Fortran code to check the type of Lua values.

- **`lua_Function` (Abstract Interface):**
    - Defines the signature for a Fortran function that can be called from Lua. It must accept a `c_ptr` (the Lua state) and return an `integer(c_int)`.

- **Public Wrapper Functions:**
    The module exposes a large number of public functions that wrap the Lua C API. These can be broadly categorized:
    - **State Management:** `fluL_newstate`, `flu_close`, `flu_isopen`, `fluL_openlibs`, `flu_copyptr`.
    - **Stack Operations:** `flu_getTop`, `flu_setTop`, `flu_pop`, `flu_insert`, `flu_pushvalue`, `flu_pushnil`, `flu_pushboolean`, `flu_pushinteger` (and variants `flu_pushint`, `flu_pushlong`), `flu_pushnumber` (and variants `flu_pushreal`, `flu_pushdouble`), `flu_pushstring`, `flu_pushlightuserdata`, `flu_pushcclosure`.
    - **Accessing Lua Variables:** `flu_getGlobal`, `flu_getField`, `flu_getTable`, `flu_rawgeti`.
    - **Setting Lua Variables:** `flu_setGlobal`, `flu_setField`, `flu_setTable`.
    - **Calling Lua Functions:** `flu_pcall`.
    - **Type Checking:** `flu_type`, `flu_isFunction`, `flu_isNumber`, `flu_isTable`, `flu_isString`, `flu_isNone`, `flu_isNoneOrNil`, `flu_isNil`, `flu_isBoolean`, `flu_islightuserdata`.
    - **Converting Lua Data to Fortran:** `flu_toboolean`, `flu_tonumber`, `flu_todouble`, `flu_tolstring`, `flu_touserdata`, `flu_topointer`.
    - **Table Manipulation:** `flu_createTable`, `flu_next`.
    - **Metatables:** `fluL_newmetatable`, `fluL_setmetatable`, `flu_getmetatable`.
    - **Loading and Running Lua Code:** `fluL_loadfile`, `fluL_loadstring`, `fluL_loadbuffer`.
    - **References:** `fluL_ref`.
    - **Registering Fortran Functions with Lua:** `flu_register`.
    - **Dumping Lua State:** `flu_dump` (interface to `flu_dump_toBuf`).
    - **Buffer Management:** `flu_free_cbuf`.

## Important Variables/Constants

- **`flu_State` type:** The cornerstone of the module, representing an individual Lua environment.
- **`FLU_T<TYPE>` constants:** Essential for determining the type of data retrieved from the Lua stack (e.g., `FLU_TSTRING`, `FLU_TNUMBER`).
- **`LUA_REGISTRYINDEX` (via `lua_fif`):** While not directly defined in `flu_binding`, it's a crucial Lua concept for storing Lua values in a place accessible from C/Fortran, often used with `fluL_ref` and `flu_rawgeti`.

## Usage Examples

```fortran
program lua_example
  use flu_binding
  use flu_kinds_module, only: int_k
  implicit none

  type(flu_State) :: L
  integer :: errcode
  character(len=256) :: message
  real(kind=c_double) :: lua_result_num
  integer(LUA_OK) :: status ! LUA_OK would come from lua_parameters or lua_fif

  ! 1. Create a new Lua state
  L = fluL_newstate()
  if (.not. flu_isopen(L)) then
    write(*,*) "Error: Could not create Lua state."
    stop 1
  end if

  ! 2. Open standard Lua libraries
  call fluL_openlibs(L)

  ! 3. Load a Lua script file
  errcode = fluL_loadfile(L, "myscript.lua")
  if (errcode /= LUA_OK) then ! Assuming LUA_OK is 0 or similar
    message = flu_tolstring(L, -1, len(message))
    write(*,*) "Error loading myscript.lua: ", trim(message)
    call flu_pop(L, 1) ! Pop error message
    call flu_close(L)
    stop 1
  end if

  ! 4. Execute the loaded script (which might define functions)
  errcode = flu_pcall(L, 0, 0, 0) ! 0 arguments, 0 results, no error function
  if (errcode /= LUA_OK) then
    message = flu_tolstring(L, -1, len(message))
    write(*,*) "Error running myscript.lua: ", trim(message)
    call flu_pop(L, 1)
    call flu_close(L)
    stop 1
  end if

  ! 5. Get and call a Lua function defined in myscript.lua
  !    Assume myscript.lua defines: function add(a, b) return a + b end
  errcode = flu_getglobal(L, "add")
  if (flu_type(L, -1) /= FLU_TFUNCTION) then
    write(*,*) "Error: 'add' is not a function in Lua script."
    call flu_pop(L, 1) ! Pop the non-function item
    call flu_close(L)
    stop 1
  end if

  ! Push arguments
  call flu_pushnumber(L, 10.0_c_double)
  call flu_pushnumber(L, 5.0_c_double)

  ! Call the function (2 arguments, 1 result)
  errcode = flu_pcall(L, 2, 1, 0)
  if (errcode /= LUA_OK) then
    message = flu_tolstring(L, -1, len(message))
    write(*,*) "Error calling Lua function 'add': ", trim(message)
    call flu_pop(L, 1)
    call flu_close(L)
    stop 1
  end if

  ! Get the result
  if (flu_isNumber(L, -1)) then
    lua_result_num = flu_todouble(L, -1)
    write(*,*) "Result of add(10, 5) from Lua: ", lua_result_num
  else
    write(*,*) "Lua function 'add' did not return a number."
  end if
  call flu_pop(L, 1) ! Pop result

  ! 6. Clean up: Close the Lua state
  call flu_close(L)

contains
  ! Example of a Fortran function to be called by Lua
  ! This would be registered using flu_register(L, "fortran_func_name", my_fortran_function)
  function my_fortran_function(s) result(val) bind(c)
    use, intrinsic :: iso_c_binding
    use flu_binding ! For flu_State, flu_tonumber, flu_pushnumber etc.
    integer(c_int) :: val
    type(c_ptr), value :: s
    type(flu_State) :: current_L

    current_L = flu_copyptr(s) ! Convert c_ptr to flu_State

    ! Get arguments from Lua stack if any
    ! e.g., local arg1 = flu_tonumber(current_L, 1)
    ! Do some computation...
    ! Push results back onto Lua stack
    ! e.g., call flu_pushnumber(current_L, result_value)
    val = 1 ! Number of return values pushed onto the stack
  end function my_fortran_function

end program lua_example
```

## Dependencies and Interactions

- **`iso_c_binding`:** Used extensively for defining interoperable types and interfaces with C.
- **`lua_fif`:** This module is fundamental as `flu_binding` acts as a higher-level wrapper around the direct C bindings likely provided by `lua_fif`. Most `flu_` functions will call a corresponding `lua_` function from `lua_fif`.
- **`lua_parameters`:** Likely provides constants and parameter definitions from the Lua C API (e.g., `LUA_OK`, `LUA_REGISTRYINDEX`).
- **`dump_lua_fif_module`:** Used to provide the `flu_dump_toBuf` functionality, which interfaces with the `dump_lua_toBuf` C function.
- **`flu_kinds_module`:** Provides kind parameters for integer types (`int_k`, `long_k`) ensuring consistent sizing.
- **Lua C Library:** The entire module is dependent on a linked Lua C library (e.g., `liblua.so` or `lua.dll`).
- **C Standard Library:** The `c_free` interface points to the standard C `free` function for memory deallocation, used by `flu_free_cbuf`.

This module serves as a critical bridge, making Lua's scripting capabilities available to Fortran applications by providing a more Fortran-friendly API layer over the raw C function calls.
