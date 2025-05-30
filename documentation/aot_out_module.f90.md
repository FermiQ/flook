# aotus/source/aot_out_module.f90

## Overview

The `aot_out_module` is designed to facilitate the generation of human-readable Lua scripts from Fortran programs. It specializes in writing various Fortran intrinsic data types (integers, reals, logicals, and strings, as both scalars and one-dimensional arrays) into a Lua script format. This includes creating Lua tables and correctly formatting the data values according to Lua syntax, managing indentation for nested structures.

This module builds upon the foundational services provided by `aot_out_general_module` (like file opening/closing and basic table structure) by adding specific procedures to handle the output of different Fortran data types. The comment "this module could stand alone, along with the flu_kinds_module without the Lua library" highlights that its output generation relies purely on Fortran I/O, not direct Lua C API calls.

## Key Components

- **Re-exported Components from `aot_out_general_module`:**
    - `aot_out_type`: The derived type for managing output state (file unit, indentation, etc.).
    - `aot_out_open`, `aot_out_close`: Subroutines for opening and closing the output destination (file or unit).
    - `aot_out_open_table`, `aot_out_close_table`: Subroutines for starting and ending Lua-style table structures.
    - `aot_out_toChunk`: Subroutine to convert the generated output in a unit to a single string.
    *(Note: `aot_out_breakline` is used internally by procedures in this module but is not re-exported publicly from `aot_out_module` itself, it's used via the dependency on `aot_out_general_module`)*

- **`aot_out_val` (Public Interface):**
  This is the primary interface provided by this module for writing Fortran data. It bundles procedures for various data types:
    - **Scalar Types:**
        - `aot_out_val_int(put_conf, val, vname, advance_previous)`
        - `aot_out_val_long(put_conf, val, vname, advance_previous)`
        - `aot_out_val_real(put_conf, val, vname, advance_previous)` (single precision)
        - `aot_out_val_double(put_conf, val, vname, advance_previous)`
        - `aot_out_val_logical(put_conf, val, vname, advance_previous)`
        - `aot_out_val_string(put_conf, val, vname, advance_previous)`
    - **1D Array Types:** These output the array as a Lua table.
        - `aot_out_val_arr_int(put_conf, val, vname, advance_previous, max_per_line)`
        - `aot_out_val_arr_long(put_conf, val, vname, advance_previous, max_per_line)`
        - `aot_out_val_arr_real(put_conf, val, vname, advance_previous, max_per_line)`
        - `aot_out_val_arr_double(put_conf, val, vname, advance_previous, max_per_line)`
        - `aot_out_val_arr_logical(put_conf, val, vname, advance_previous, max_per_line)`
        - `aot_out_val_arr_string(put_conf, val, vname, advance_previous, max_per_line)`
    Common optional arguments:
        - `vname`: A string to assign the value to a Lua variable (e.g., `vname = value`).
        - `advance_previous`: Controls line breaking before this item (default is true, start on new line).
        - `max_per_line` (for arrays): Suggests how many array elements to print on a single line within the Lua table.
    *(The module also `use`s `aot_quadruple_out_module` and `aot_extdouble_out_module`. This implies that the `aot_out_val` interface is extended by procedures from these modules to support outputting quadruple precision and other extended double precision real types, following similar naming conventions.)*

## Important Variables/Constants

- **`put_conf (type(aot_out_type))`**: The handle managing the output state, passed to all procedures.
- **Formatting within procedures**:
    - Logical values are written as `true` or `false`.
    - Strings are enclosed in single quotes (e.g., `'my string'`).
    - Real numbers are formatted using specific Fortran format descriptors (e.g., `f16.7` for single, `EN24.15` for double).
    - Array output routines (`aot_out_val_arr_*`) internally use `aot_out_open_table`, then loop calling the scalar `aot_out_val` for each element, and finally `aot_out_close_table`.

## Usage Examples

```fortran
program generate_lua_config
  use aot_out_module
  use flu_kinds_module, only: dk=>double_k, ik=>int_k, lk=>long_k
  implicit none

  type(aot_out_type) :: config_writer
  integer(ik) :: int_scalar, int_array(3)
  real(dk) :: dbl_scalar, dbl_array(2)
  logical :: log_val, log_array(2)
  character(len=80) :: str_val, str_array(2)

  ! Initialize data
  int_scalar = 123
  int_array = [1, 2, 3]
  dbl_scalar = 456.789_dk
  dbl_array = [1.1_dk, 2.2_dk]
  log_val = .true.
  log_array = [.true., .false.]
  str_val = "Hello Lua"
  str_array = ["FirstString", "SecondString"]

  ! Open output file
  call aot_out_open(config_writer, filename="generated_script.lua", indentation=2)

  ! Write a global variable
  call aot_out_val(config_writer, int_scalar, vname="my_global_int")

  ! Start a main table
  call aot_out_open_table(config_writer, tname="settings")

  ! Write various scalar values into the table
  call aot_out_val(config_writer, dbl_scalar, vname="double_value")
  call aot_out_val(config_writer, log_val, vname="boolean_flag")
  call aot_out_val(config_writer, str_val, vname="greeting_string")
  call aot_out_val(config_writer, 9876543210_lk, vname="long_integer_value")

  ! Write an integer array as a named table field
  call aot_out_val(config_writer, int_array, vname="integer_list", max_per_line=5)

  ! Write a double array as a named table field
  call aot_out_val(config_writer, dbl_array, vname="double_list", max_per_line=2)

  ! Write a logical array
  call aot_out_val(config_writer, log_array, vname="boolean_list", max_per_line=4)

  ! Write a string array
  call aot_out_val(config_writer, str_array, vname="string_list", max_per_line=1)

  ! Close the main table
  call aot_out_close_table(config_writer)

  ! Close the output file
  call aot_out_close(config_writer)

  write(*,*) "Generated generated_script.lua"

end program generate_lua_config
```

This would produce a `generated_script.lua` file looking something like:

```lua
my_global_int = 123
settings = {
  double_value =   4.567890000000000E+02,
  boolean_flag = true,
  greeting_string = 'Hello Lua',
  long_integer_value = 9876543210,
  integer_list = {
    1, 2, 3
  },
  double_list = {
    1.100000000000000E+00, 2.200000000000000E+00
  },
  boolean_list = {
    true, false
  },
  string_list = {
    'FirstString',
    'SecondString'
  }
}
```

## Dependencies and Interactions

- **`flu_kinds_module`**: Essential for defining the kinds of numerical data (`double_k`, `single_k`, `int_k`, `long_k`) that the `aot_out_val` procedures can handle.
- **`aot_out_general_module`**: This module forms the backbone for `aot_out_module`. `aot_out_module` re-exports and uses the `aot_out_type` and fundamental routines for opening/closing files/units (`aot_out_open`, `aot_out_close`), managing table structures (`aot_out_open_table`, `aot_out_close_table`), and converting output to a string (`aot_out_toChunk`). The `aot_out_breakline` from the general module is used internally by the `aot_out_val_*` procedures.
- **`aot_quadruple_out_module` & `aot_extdouble_out_module`**: These modules are `use`d, indicating that the `aot_out_val` interface is extended to support outputting these higher-precision real types through specific procedures likely defined within them.
- **Lua Environment (Indirect):** The output generated by this module is intended to be valid Lua code, meaning it's designed to be parsed and executed by a Lua interpreter.

`aot_out_module` provides a convenient Fortran-idiomatic way to serialize various data structures into Lua script format, which can be used for configuration files, data exchange, or generating dynamic Lua code.
