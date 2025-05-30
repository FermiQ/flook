# aotus/LuaFortran/lua_fif.f90

## Overview

This module, `lua_fif` (Lua Fortran Interface File), provides a direct, low-level interface from Fortran to a subset of the Lua 5.3.2 C Application Programming Interface (API). It utilizes the `iso_c_binding` intrinsic module to define Fortran interfaces that correspond to C functions in the Lua library. This module serves as the foundational communication layer, enabling Fortran programs to call core Lua functionalities. It is typically not used directly for application development but rather as a base for higher-level Fortran wrapper modules, such as `flu_binding`.

## Key Components

The module consists of a series of Fortran `interface` blocks. Each block defines a Fortran subroutine or function that binds to a specific C function from the Lua API. These interfaces allow Fortran code to:

- **Manage the Lua State:**
    - `lua_close(L)`: Closes a Lua state.
    - `luaL_newstate()`: Creates a new Lua state.
    - `luaL_openlibs(L)`: Opens standard Lua libraries into a state.

- **Manipulate the Lua Stack:**
    - `lua_gettop(L)`: Gets the index of the top element on the stack.
    - `lua_settop(L, index)`: Sets the stack top to a specific index (or pops elements).
    - `lua_pushnil(L)`, `lua_pushboolean(L, n)`, `lua_pushinteger(L, n)`, `lua_pushnumber(L, n)`, `lua_pushlstring(L, s, len)`, `lua_pushvalue(L, index)`: Push various types onto the stack.
    - `lua_rotate(L, idx, n)`: Rotates stack elements.

- **Access Lua Variables and Table Elements:**
    - `lua_getglobal(L, k)`: Pushes the value of a global variable `k` onto the stack.
    - `lua_getfield(L, index, k)`: Pushes the value of `t[k]` (where `t` is at `index`) onto the stack.
    - `lua_gettable(L, index)`: Pushes the value of `t[key]` (where `t` is at `index` and `key` is at top of stack) onto the stack.
    - `lua_rawgeti(L, index, n)`: Pushes the value of `t[n]` (where `t` is at `index`) onto the stack (raw access).

- **Set Lua Variables and Table Elements:**
    - `lua_setglobal(L, k)`: Pops a value from the stack and sets it as global variable `k`.
    - `lua_setfield(L, index, k)`: Pops a value and sets it as `t[k]` (where `t` is at `index`).
    - `lua_settable(L, index)`: Pops key and value, and sets `t[key] = value` (where `t` is at `index`).

- **Execute Lua Code:**
    - `lua_pcallk(L, nargs, nresults, errfunc, ctx, k)`: Calls a Lua function in protected mode.
    - `luaL_loadfilex(L, filename, mode)`: Loads a Lua file.
    - `luaL_loadbufferx(L, buff, sz, name, mode)`: Loads Lua code from a buffer.
    - `luaL_loadstring(L, string)`: Loads Lua code from a string.

- **Query and Convert Types:**
    - `lua_type(L, index)`: Returns the type of the value at `index`.
    - `lua_isNumber(L, index)`, `lua_isString(L, index)`: Check if a value at `index` is of a specific type.
    - `lua_toboolean(L, index)`, `lua_tonumberx(L, index, isnum)`, `lua_tolstring(L, index, len)`, `lua_touserdata(L, index)`, `lua_topointer(L, index)`: Convert Lua values at `index` to Fortran/C types.

- **Table Operations:**
    - `lua_createtable(L, narr, nrec)`: Creates a new empty table on the stack.
    - `lua_next(L, index)`: Pops a key and pushes a key-value pair from the table at `index`.

- **Metatables:**
    - `lua_getmetatable(L, index)`: Pushes the metatable of the value at `index`.
    - `luaL_newmetatable(L, tname)`: Creates a new metatable in the registry.
    - `luaL_setmetatable(L, tname)`: Sets the metatable for the object at the top of the stack.

- **Userdata and C Functions:**
    - `lua_pushlightuserdata(L, ptr)`: Pushes a light C userdata onto the stack.
    - `lua_pushcclosure(L, c_fn, n)`: Pushes a new C closure (Fortran function callable by Lua) onto the stack.

- **References:**
    - `luaL_ref(L, t)`: Creates and returns a reference in the table at index `t`.

All functions take a `type(c_ptr), value :: L` argument, representing the Lua state. String arguments are passed as `character(kind=c_char), dimension(*)`, and many integer arguments/results are `integer(kind=c_int)`.

## Important Variables/Constants

- **`type(c_ptr) :: L`**: This is the fundamental argument in all interface calls, representing the pointer to the opaque Lua state structure (`lua_State*` in C).
- **`lua_parameters` module**: This module is `use`d and is expected to provide:
    - Kind parameters for Lua numbers and integers (e.g., `lua_num`, `lua_int`).
    - Integer constants representing Lua status codes (e.g., `LUA_OK`, `LUA_ERRRUN`) and special Lua registry indices (e.g., `LUA_REGISTRYINDEX`). These are crucial for checking return values and interacting with the Lua registry.

## Usage Examples

This module provides direct, low-level bindings. For more convenient, Fortran-idiomatic usage, these interfaces are typically wrapped by higher-level modules like `flu_binding.f90`. Direct usage would closely mirror C API calls:

```fortran
! Conceptual direct usage (usually done via flu_binding)
module main_prog
  use lua_fif
  use lua_parameters ! For LUA_OK, lua_int, etc.
  use, intrinsic :: iso_c_binding
  implicit none

  type(c_ptr) :: L
  integer(c_int) :: error_status
  character(len=256, kind=c_char) :: script_name_c
  real(lua_num) :: num_result
  integer(c_int) :: isnum_flag

  L = luaL_newstate()
  if (.not. c_associated(L)) then
    write(*,*) "Cannot create Lua state"
    stop
  end if

  call luaL_openlibs(L)

  script_name_c = "my_script.lua" // c_null_char
  error_status = luaL_loadfilex(L, script_name_c, c_null_char) ! mode ""

  if (error_status /= LUA_OK) then
    ! Error handling: get error message from stack
    ! ... (using lua_tolstring)
    call lua_close(L)
    stop
  end if

  error_status = lua_pcallk(L, 0, 0, 0, 0, c_null_ptr) ! Call loaded script
  if (error_status /= LUA_OK) then
    ! Error handling
    ! ...
    call lua_close(L)
    stop
  end if

  ! Assume script put a global number 'my_result'
  error_status = lua_getglobal(L, "my_result" // c_null_char)
  if (error_status == LUA_TNUMBER) then ! LUA_TNUMBER from lua_parameters
     num_result = lua_tonumberx(L, -1, isnum_flag)
     if (isnum_flag /= 0) then
        write(*,*) "Result:", num_result
     end if
  end if
  call lua_settop(L, -2) ! Pop result or nil

  call lua_close(L)

end module main_prog
```

## Dependencies and Interactions

- **`iso_c_binding` (Intrinsic Module):** This is the core Fortran feature enabling the definition of C-interoperable interfaces.
- **`lua_parameters` (Module):** This module is crucial as it supplies the necessary kind parameters for Lua-specific number (`lua_num`) and integer (`lua_int`) types, as well as important constants from `lua.h` (like `LUA_OK`, `LUA_REGISTRYINDEX`, various type tags `LUA_T...`, etc.) that are essential for working with the Lua API.
- **External Lua C Library (e.g., `liblua.so`, `lua.dll`):** The Fortran interfaces defined in this module bind to actual C functions provided by an external Lua library. The application must be linked against a Lua 5.3.2 compatible library for these bindings to resolve and function correctly.
- **`flu_binding.f90` (Module):** `lua_fif.f90` serves as the foundational layer for `flu_binding.f90`. The `flu_binding` module takes these direct C interfaces and wraps them in more Fortran-friendly subroutines and functions, often handling C string conversions, type conversions, and providing a more abstracted API (`flu_` prefixed functions).

This module is essential for any Fortran project aiming to interface with Lua at a low level, forming the bridge over which all communication with the Lua C library passes.
