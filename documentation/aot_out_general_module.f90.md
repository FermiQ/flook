# aotus/source/aot_out_general_module.f90

## Overview

The `aot_out_general_module` provides a suite of tools for generating formatted text output, primarily designed to create content that resembles Lua script syntax. This includes support for nested tables, assignments, and proper indentation. The module can write to a specified file or a pre-opened Fortran unit. A key feature is the ability to capture the generated output as a single string (chunk), which can be useful for dynamically creating Lua code to be executed.

It manages the state of the output process, including indentation levels and the nesting of table structures, to ensure the generated output is well-formed and readable.

## Key Components

- **`aot_out_type` (Derived Type):**
  This type encapsulates the state of an ongoing output operation.
    - `outunit (integer)`: The Fortran unit number to which output is directed.
    - `indent (integer)`: The current indentation level (number of spaces).
    - `stack(100) (integer)`: An array storing the number of entries at each nesting level of tables, used for separator logic.
    - `level (integer)`: The current nesting level (e.g., how many tables are currently open).
    - `externalOpen (logical)`: A flag indicating if `outunit` was opened outside this module's scope (true) or by `aot_out_open` (false).
    - `in_step (integer)`: The number of spaces to use for each level of indentation (defaults to 4).

- **`aot_out_open(put_conf, filename, outUnit, indentation)` (Subroutine):**
  Initializes an `aot_out_type` variable (`put_conf`).
    - If `filename` is provided, it opens the specified file for writing (replacing it if it exists) and associates it with a new unit.
    - If `outUnit` is provided (and `filename` is not), it uses the pre-connected Fortran unit.
    - `indentation` (optional) sets the number of spaces per indentation level.

- **`aot_out_close(put_conf)` (Subroutine):**
  Closes the Fortran unit associated with `put_conf`, but only if it was opened by `aot_out_open` (i.e., `put_conf%externalOpen` is false).

- **`aot_out_open_table(put_conf, tname, advance_previous)` (Subroutine):**
  Writes the beginning of a Lua-style table to the output unit.
    - `tname` (optional): If provided, writes `tname = {`. Otherwise, writes `{`.
    - `advance_previous` (optional, default true): If true, starts the table on a new line. If false, appends to the current line.
    - Increments the indentation level.

- **`aot_out_close_table(put_conf, advance_previous)` (Subroutine):**
  Writes the closing brace `}` for the current Lua-style table.
    - `advance_previous` (optional, default true): If true, writes the closing brace on a new line. If false, appends ` }` to the current line.
    - Decrements the indentation level and resets the entry count for the closed level.

- **`aot_out_breakline(put_conf, advance_previous)` (Subroutine):**
  Manages line endings and separators (`,`) for entries within a table or between statements.
    - `advance_previous` (optional, default true): If true, the next entry will start on a new, indented line. If false, a space is added for same-line entries.
    - Automatically adds a comma separator if needed based on the number of items already written at the current table level.

- **`aot_out_toChunk(out_conf, chunk, ErrCode, ErrString)` (Subroutine):**
  Reads all content previously written to the Fortran unit associated with `out_conf`, concatenates it line by line (with newlines) into the output string `chunk`.
    - `chunk (character(len=*), intent(out))`: The string to hold the full output.
    - `ErrCode`, `ErrString` (optional): For error reporting during the read process (e.g., unit not open, chunk too small). If not provided, stops on error.

- **`newunit() result(nu)` (Function):**
  A utility function to find an available Fortran I/O unit number. It starts checking from unit 21 upwards. This is a workaround for older Fortran compilers that may not support the `NEWUNIT=` specifier in the `OPEN` statement.

## Important Variables/Constants

- The components of the `aot_out_type` are critical for managing the state of the output generation process, particularly `outunit`, `indent`, `level`, and `stack`.
- `put_conf%in_step` controls the visual appearance of the indentation.

## Usage Examples

**1. Generating a simple Lua-like table to a file:**

```fortran
program test_aot_out
  use aot_out_general_module
  implicit none

  type(aot_out_type) :: config_output
  character(len=80) :: entry_name
  integer :: i

  ! Open a file for output
  call aot_out_open(config_output, filename="my_config.lua", indentation=2)

  ! Start a named table
  call aot_out_open_table(config_output, tname="my_settings")

  ! Write some key-value pairs (conceptual - actual value writing needs more)
  ! aot_out_breakline would be called before writing each "key = value" part
  do i = 1, 3
     call aot_out_breakline(config_output) ! Start new line, add comma if needed
     write(config_output%outunit, fmt='(A, A, I0, A)', advance='no') &
       & "key", "_", i, " = "
     ! (Here you would write the actual value, e.g., using another write statement)
     write(config_output%outunit, fmt='(I0)', advance='no') i * 10
  end do

  ! Close the table
  call aot_out_close_table(config_output)

  ! Close the file
  call aot_out_close(config_output)

  write(*,*) "Generated my_config.lua"

end program test_aot_out
```

**2. Generating Lua code into a string (chunk):**

```fortran
program test_aot_out_chunk
  use aot_out_general_module
  implicit none

  type(aot_out_type) :: string_output_conf
  character(len=1024) :: lua_script_chunk
  integer :: temp_unit, err_c
  character(len=256) :: err_s

  ! Get a new unit and open it temporarily for internal use
  temp_unit = newunit()
  open(unit=temp_unit, status='scratch', action='readwrite')

  ! Initialize aot_out_type with the pre-opened unit
  call aot_out_open(string_output_conf, outUnit=temp_unit)

  ! Use output routines to write Lua-like content
  call aot_out_open_table(string_output_conf, tname="data")
  call aot_out_breakline(string_output_conf)
  write(string_output_conf%outunit, fmt='(A)', advance='no') "value = 123"
  call aot_out_close_table(string_output_conf)

  ! Convert the content of the unit to a string
  call aot_out_toChunk(string_output_conf, lua_script_chunk, ErrCode=err_c, ErrString=err_s)

  if (err_c == 0) then
    write(*,*) "Generated Lua Chunk:"
    write(*,*) trim(lua_script_chunk)
  else
    write(*,*) "Error creating chunk: ", trim(err_s)
  end if

  ! Close the temporary unit (aot_out_close won't as it was externalOpen)
  close(temp_unit)

end program test_aot_out_chunk
```

## Dependencies and Interactions

- **Fortran Intrinsic I/O:** The module heavily relies on standard Fortran `WRITE`, `OPEN`, `CLOSE`, `INQUIRE`, and `REWIND` statements for its operations.
- **No other AOTUS modules:** This module appears to be largely self-contained regarding dependencies on other custom modules within the AOTUS framework for its core logic.
- **Lua Environment (Indirect):** While it doesn't directly link to or call Lua, the output it generates is formatted to be syntactically compatible with Lua scripts. The generated files or chunks are intended to be loaded or processed by a Lua interpreter.

This module is a utility for constructing Lua script content programmatically from Fortran, which can be useful for generating configuration files, dynamic scripts, or serializing Fortran data into a Lua-readable format.
