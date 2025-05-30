# aotus/source/aot_table_module.f90

## Overview

The `aot_table_module` offers a rich set of Fortran-friendly procedures for interacting with Lua tables. It simplifies common tasks such as retrieving values of various Fortran data types from Lua tables (or global variables), setting values within Lua tables, checking for the existence of table elements, and creating new Lua tables directly from one-dimensional Fortran arrays.

This module builds upon lower-level operations in `aot_table_ops_module` and `flu_binding` to provide a more abstracted and convenient API for developers. It also incorporates error reporting using codes defined in `aot_err_module`.

## Key Components

- **Re-exported Procedures from `aot_table_ops_module`:**
  This module re-exports several fundamental table operations, making them directly available to users of `aot_table_module`:
    - `aot_table_top`: Gets a handle to a table at the top of the Lua stack.
    - `aot_table_length`: Gets the length of a table (Lua's `#` operator).
    - `aot_table_first`: Pushes the first key-value pair of a table onto the stack for iteration.
    - `aot_table_push`: Pushes a specific element from a table (by key or position) or a global variable onto the stack.
    - `aot_table_open`: Opens a table (by key or global name) and returns its handle (stack index).
    - `aot_table_close`: Closes a table handle (pops it from the stack).
    - `aot_type_of`: Determines the Lua type of an element within a table or a global variable.

- **`aot_table_get_val` (Interface, also available as `aot_get_val`):**
  A generic interface for retrieving scalar values from Lua. If `thandle` (table handle/stack index) is provided, it fetches from the table using `key` or `pos`. If `thandle` is absent, it fetches a global variable using `key`.
  Specific procedures include:
    - `get_table_real(val, ErrCode, L, thandle, key, pos, default)` (single precision)
    - `get_table_double(val, ErrCode, L, thandle, key, pos, default)`
    - `get_table_integer(val, ErrCode, L, thandle, key, pos, default)`
    - `get_table_long(val, ErrCode, L, thandle, key, pos, default)`
    - `get_table_string(val, ErrCode, L, thandle, key, pos, default)`
    - `get_table_logical(val, ErrCode, L, thandle, key, pos, default)`
    - `get_table_userdata(val, ErrCode, L, thandle, key, pos, default)` (retrieves a `c_ptr`)
  These routines output the retrieved `val` and an `ErrCode` (see `aot_err_module`). An optional `default` value can be provided.

- **`aot_table_set_val` (Interface):**
  A generic interface for setting scalar values within a Lua table specified by `thandle`. The element is identified by `key` (preferred) or `pos`.
  Specific procedures include:
    - `set_table_real(val, L, thandle, key, pos)`
    - `set_table_double(val, L, thandle, key, pos)`
    - `set_table_integer(val, L, thandle, key, pos)`
    - `set_table_long(val, L, thandle, key, pos)`
    - `set_table_string(val, L, thandle, key, pos)`
    - `set_table_logical(val, L, thandle, key, pos)`
    - `set_table_userdata(val, L, thandle, key, pos)` (sets a `c_ptr` as light userdata)

- **`aot_table_set_top(L, thandle, key, pos)` (Subroutine):**
  Takes the Lua value currently on top of the stack and sets it as an element in the table `thandle` at the specified `key` or `pos`.

- **`aot_table_from_1Darray` (Interface):**
  Creates a new Lua table (numerically indexed) from a 1D Fortran array.
  Specific procedures include:
    - `create_1Darray_real(L, thandle, val)` (single precision)
    - `create_1Darray_double(L, thandle, val)`
  These return a `thandle` (stack index) to the newly created Lua table.

- **`aot_exists(L, thandle, key, pos) result(exists)` (Function):**
  Checks if a Lua entity (specified by `key` or `pos` within `thandle`, or as a global if `thandle` is absent) exists and is not nil. Returns a logical `exists`.

- **Support for Higher Precision:**
  The module `use`s `aot_quadruple_table_module` and `aot_extdouble_table_module`. This indicates that the interfaces `aot_table_get_val`, `aot_table_set_val`, and `aot_table_from_1Darray` are likely extended with specific procedures from these modules to handle quadruple precision and other extended double precision real numbers.

## Important Variables/Constants

- **`L (type(flu_State))`**: The Lua state handle, required by all procedures.
- **`thandle (integer)`**: Typically an input argument representing the stack index of the Lua table to be operated upon. For `aot_table_from_1Darray`, it's an output handle to the new table.
- **`key (character(len=*))`**: Optional string argument to identify table elements by name.
- **`pos (integer)`**: Optional integer argument to identify table elements by numerical index.
- **`val`**: The Fortran variable to receive a value from Lua or provide a value to be set in Lua. Its type matches the specific procedure called.
- **`ErrCode (integer)`**: Output from `get_table_*` routines, indicating success or specific errors (e.g., `aoterr_NonExistent`, `aoterr_WrongType`) using bit flags from `aot_err_module`.
- **`default`**: Optional argument in `get_table_*` routines; if the Lua value is not found or is of the wrong type, `val` is set to this default.

## Usage Examples

```fortran
program test_aot_table
  use aot_table_module
  use flu_binding       ! For L, fluL_newstate, fluL_openlibs, fluL_dostring, flu_close
  use flu_kinds_module  ! For kind parameters like dk, ik
  use aot_err_module    ! For error constants like aoterr_NonExistent
  implicit none

  type(flu_State) :: L
  integer :: main_table_h, sub_table_h, err_code
  real(dk) :: d_val
  integer(ik) :: i_val, i_array_fort(3)
  logical :: exists_flag

  L = fluL_newstate()
  call fluL_openlibs(L)

  ! Create some Lua tables and values for testing
  call fluL_dostring(L, "main_config = { version = 1.2, " // &
    & " sub_settings = { id = 101, name = 'test_device' }, " // &
    & " data_points = {10, 20, 30} }")

  ! 1. Open the global 'main_config' table
  call aot_table_open(L, main_table_h, key="main_config")
  if (main_table_h == 0) then
    write(*,*) "Failed to open main_config table."
    call flu_close(L)
    stop
  end if

  ! 2. Get a value from 'main_config'
  call aot_get_val(d_val, err_code, L, main_table_h, key="version", default=-1.0_dk)
  if (btest(err_code, aoterr_NonExistent)) then
    write(*,*) "main_config.version not found, using default: ", d_val
  else
    write(*,*) "main_config.version: ", d_val
  end if

  ! 3. Open the nested 'sub_settings' table
  call aot_table_open(L, sub_table_h, main_table_h, key="sub_settings")

  ! 4. Get a value from 'sub_settings'
  call aot_get_val(i_val, err_code, L, sub_table_h, key="id")
  write(*,*) "main_config.sub_settings.id: ", i_val

  ! 5. Set a new value in 'sub_settings'
  call aot_table_set_val(L, sub_table_h, key="new_flag", val=.true.)

  ! 6. Check if 'new_flag' exists in 'sub_settings'
  exists_flag = aot_exists(L, sub_table_h, key="new_flag")
  write(*,*) "main_config.sub_settings.new_flag exists: ", exists_flag

  call aot_table_close(L, sub_table_h) ! Close sub_settings

  ! 7. Create a new Lua table from a Fortran array and add it to main_config
  i_array_fort = [77_ik, 88_ik, 99_ik]
  call aot_table_from_1Darray(L, sub_table_h, i_array_fort) ! sub_table_h now points to the new array table
  call aot_table_set_top(L, main_table_h, key="fortran_array") ! Add it to main_config

  call aot_table_close(L, main_table_h) ! Close main_config

  ! ... (verify by printing the modified main_config table from Lua, if desired) ...

  call flu_close(L)

end program test_aot_table
```

## Dependencies and Interactions

- **`flu_binding`**: Provides the low-level Lua C API bindings (e.g., `flu_State`, `flu_getglobal`, stack manipulation for pushing/setting values).
- **`flu_kinds_module`**: Supplies Fortran kind parameters (`double_k`, `single_k`, `int_k`, `long_k`) used in procedure interfaces and variable declarations.
- **`aot_err_module`**: Defines error constants (`aoterr_Fatal`, `aoterr_NonExistent`, `aoterr_WrongType`) used in the `ErrCode` output of `get_table_*` routines.
- **`aot_top_module`**: The `aot_top_get_val` subroutine from this module is used internally by the `get_table_*` procedures in `aot_table_module` to retrieve values from the top of the Lua stack once they've been pushed there.
- **`aot_table_ops_module`**: This is a significant underlying module. `aot_table_module` re-exports many of its core table operation routines (like `aot_table_open`, `aot_table_close`, `aot_table_push`) and builds upon them to offer more specialized value-typed getters and setters.
- **`aot_quadruple_table_module` & `aot_extdouble_table_module`**: These modules are `use`d to extend the functionality of `aot_table_module`'s interfaces (`aot_table_get_val`, `aot_table_set_val`, `aot_table_from_1Darray`) for handling quadruple precision and other extended double precision real numbers.

`aot_table_module` serves as a key utility for bridging Fortran data with Lua tables, facilitating data exchange and configuration management in AOTUS.
