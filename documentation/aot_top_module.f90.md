# aotus/source/aot_top_module.f90

## Overview

The `aot_top_module` provides essential routines for interacting with the Lua value located at the top of the Lua execution stack. Its primary function is to retrieve this topmost value and convert it into various standard Fortran data types (real, integer, logical, string, and C pointers via userdata).

This module is fundamental for obtaining results from Lua operations that leave their return values on the stack (e.g., after calling a Lua function or explicitly fetching a global/table field onto the stack). It includes robust error handling, allowing the caller to specify default values if a retrieval fails or if the Lua type is incompatible. After attempting retrieval, the routines in this module will pop the value from the Lua stack.

## Key Components

- **Re-exported Components from `aot_err_module`:**
  This module re-exports key error-handling components, making them conveniently available:
    - `aoterr_Fatal`, `aoterr_NonExistent`, `aoterr_WrongType`: Integer parameter constants representing different error conditions.
    - `aot_err_handler`: The general Lua error handling subroutine.

- **`aot_top_get_val` (Public Interface):**
  This is the core generic interface of the module, designed to retrieve and convert the value at the top of the Lua stack into a specified Fortran type. All procedures within this interface pop the value from the stack after processing.
  Specific procedures include:
    - **`aot_top_get_real(val, ErrCode, L, default)`**: Retrieves a single-precision real.
    - **`aot_top_get_double(val, ErrCode, L, default)`**: Retrieves a double-precision real.
    - **`aot_top_get_integer(val, ErrCode, L, default)`**: Retrieves a default-kind integer (converted from Lua number).
    - **`aot_top_get_long(val, ErrCode, L, default)`**: Retrieves a long-kind integer (converted from Lua number).
    - **`aot_top_get_string(val, ErrCode, L, default)`**: Retrieves a character string.
    - **`aot_top_get_logical(val, ErrCode, L, default)`**: Retrieves a logical value.
    - **`aot_top_get_userdata(val, ErrCode, L, default)`**: Retrieves a light userdata as a `type(c_ptr)`.

  **Common Arguments for `aot_top_get_*` procedures:**
    - `val (type, intent(out))`: The Fortran variable that will store the retrieved value. Its type corresponds to the specific procedure called (e.g., `real(kind=single_k)` for `aot_top_get_real`).
    - `ErrCode (integer, intent(out))`: An error code indicating the outcome. It's a bitfield set using constants like `aoterr_NonExistent` or `aoterr_WrongType`. `0` usually means success.
    - `L (type(flu_State))`: The Lua state handle.
    - `default (type, optional, intent(in))`: An optional value of the same type as `val`. If the Lua value cannot be retrieved (e.g., it's nil, or the type is wrong) and `default` is provided, `val` is set to this default value. If `default` is not provided in such cases, `ErrCode` will also typically have `aoterr_Fatal` set.

- **Support for Higher Precision:**
  The module `use`s `aot_quadruple_top_module` and `aot_extdouble_top_module`. This indicates that the `aot_top_get_val` interface is likely extended by specific procedures from these modules to support retrieving quadruple precision and other extended double precision real numbers from the Lua stack (though Lua's native number type is typically double precision).

## Important Variables/Constants

- **`L (type(flu_State))`**: The Lua state handle, passed to all retrieval routines.
- **`ErrCode (integer)`**: The primary mechanism for error reporting. It's constructed by setting bits corresponding to `aoterr_NonExistent`, `aoterr_WrongType`, and `aoterr_Fatal` (from `aot_err_module`).
- **Stack Behavior**: A critical characteristic of all `aot_top_get_*` procedures is that they **pop one value from the Lua stack** after attempting to retrieve and convert it. Callers must ensure a value is indeed on the stack before calling these routines.

## Usage Examples

```fortran
program test_aot_top_retrieval
  use aot_top_module
  use flu_binding      ! For L, fluL_newstate, fluL_openlibs, flu_getglobal, flu_close etc.
  use flu_kinds_module ! For kind parameters like dk, ik
  implicit none

  type(flu_State) :: L
  real(dk) :: d_val
  integer(ik) :: i_val
  character(len=100) :: s_val
  logical :: b_val
  integer :: err_c, lua_type

  L = fluL_newstate()
  call fluL_openlibs(L)

  ! Example 1: Get a global number
  lua_type = flu_getglobal(L, "math") ! Pushes the 'math' table
  lua_type = flu_getfield(L, -1, "pi") ! Pushes math.pi (a number)
                                     ! Stack: math_table, pi_value
  call aot_top_get_val(d_val, err_c, L, default=0.0_dk) ! Retrieves pi_value, pops it
                                                       ! Stack: math_table
  if (err_c == 0) then
    write(*,*) "Successfully retrieved math.pi: ", d_val
  else
    write(*,*) "Error retrieving math.pi, ErrCode: ", err_c
  end if
  call flu_pop(L,1) ! Pop math_table

  ! Example 2: Get a global string, provide default
  lua_type = flu_getglobal(L, "my_string_variable") ! Pushes nil if it doesn't exist
  call aot_top_get_val(s_val, err_c, L, default="DefaultString")
  write(*,*) "Retrieved string: '", trim(s_val), "', ErrCode: ", err_c
  ! s_val will be "DefaultString" if my_string_variable was nil or not a string.

  ! Example 3: Get a boolean after a function call (conceptual)
  ! Assume a Lua function "isValid()" is called and leaves a boolean on stack.
  ! call flu_pcall(...) ! This would leave result(s) on stack
  ! If isValid() returned true:
  call flu_pushboolean(L, .true.) ! Simulate function return for example
  call aot_top_get_val(b_val, err_c, L)
  if (err_c == 0) then
    write(*,*) "Retrieved boolean: ", b_val
  else
    write(*,*) "Error retrieving boolean, ErrCode: ", err_c
  end if

  call flu_close(L)

end program test_aot_top_retrieval
```

## Dependencies and Interactions

- **`flu_binding`**: This is the primary dependency. `aot_top_module` relies heavily on `flu_` prefixed functions from `flu_binding` for all direct Lua C API interactions, such as:
    - Type checking on the stack (`flu_isNoneOrNil`, `flu_isNumber`, `flu_isBoolean`, `flu_isString`, `flu_islightuserdata`).
    - Converting Lua types to Fortran types (`flu_toNumber`, `flu_toDouble`, `flu_toBoolean`, `flu_toLString`, `flu_touserdata`).
    - Stack manipulation (`flu_pop`).
- **`flu_kinds_module`**: Provides Fortran kind parameters (`double_k`, `single_k`, `int_k`, `long_k`) used in the interfaces of the retrieval routines and for declaring compatible Fortran variables.
- **`aot_err_module`**: This module is used for its error constants (`aoterr_Fatal`, `aoterr_NonExistent`, `aoterr_WrongType`), which are used with `ibSet` to construct the `ErrCode` returned by the `aot_top_get_*` procedures. The `aot_err_handler` is also re-exported from this module.
- **`aot_quadruple_top_module` & `aot_extdouble_top_module`**: These modules are `use`d, indicating that the `aot_top_get_val` interface is extended by specific procedures from these modules to support retrieving higher-precision real numbers.
- **Other AOTUS Modules**: Modules like `aot_table_module` (specifically its `get_table_*` routines) and `aot_fun_module` (when retrieving function results) depend on `aot_top_module`. They first arrange for the desired Lua value to be on top of the stack (e.g., using `aot_table_push` or after `flu_pcall`) and then call upon `aot_top_get_val` to perform the actual retrieval and type conversion into a Fortran variable.

`aot_top_module` is a crucial low-level utility in the AOTUS framework for the final step of bringing Lua data into the Fortran environment from the Lua stack.
