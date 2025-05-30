# aotus/source/aot_table_ops_module.f90

## Overview

The `aot_table_ops_module` provides a set of general, low-level operations for interacting with Lua tables from Fortran. These procedures are foundational and handle the core mechanics of table access, such as opening and closing tables, pushing table elements onto the Lua stack, determining element types, and initiating table iteration.

These operations are generally type-agnostic and serve as the building blocks for more specialized, type-specific table interaction routines found in other modules (like `aot_table_module`).

## Key Components

- **`aot_table_top(L) result(thandle)` (Function):**
  Checks if the item at the top of the Lua stack (`L`) is a table.
    - If it is a table, returns its stack index as `thandle`.
    - If not a table, pops the item and returns `thandle = 0`.

- **`aot_table_open(L, parent, thandle, key, pos)` (Subroutine):**
  Opens a Lua table and provides its stack index as `thandle`.
    - If `parent` (stack index of an existing table) is provided, it attempts to open a sub-table identified by `key` (string name) or `pos` (integer index).
    - If only `key` is provided (no `parent`), it attempts to open a global table with that name.
    - If neither `parent`, `key`, nor `pos` is provided, it creates a new, empty table on the stack.
    - If the target is not a table or doesn't exist, `thandle` is set to 0.

- **`aot_table_close(L, thandle)` (Subroutine):**
  "Closes" an opened table referenced by `thandle`. This is achieved by setting the Lua stack top to the position just below the table, effectively removing the table and any elements pushed above it from direct Lua stack accessibility.

- **`aot_table_push(L, thandle, key, pos, toptype)` (Subroutine, also available via generic interface `aot_push`):**
  Pushes a Lua value onto the top of the Lua stack.
    - If `thandle` is provided, it pushes the element `thandle[key]` or `thandle[pos]`. If both `key` and `pos` are given, `key` is tried first. If the element is not found, `nil` is pushed.
    - If only `key` is provided (no `thandle`), it pushes the global variable `key`. If not found, `nil` is pushed.
    - If only `pos` is provided (no `thandle`), this is considered an illegal operation, and `nil` is pushed.
    - If none of `thandle`, `key`, or `pos` are provided, the subroutine effectively does nothing to change the stack but will report the type of the current top element in `toptype`.
    - `toptype` (optional, intent(out)): Returns the Lua type (e.g., `FLU_TSTRING`, `FLU_TNIL`) of the value that was pushed (or was already at the top).

- **`aot_type_of(L, thandle, key, pos) result(luatype)` (Function):**
  Determines and returns the Lua type of a specified Lua entity.
    - If `thandle` is provided, it checks the element `thandle[key]` or `thandle[pos]`.
    - If only `key` is provided, it checks the global variable `key`.
    - If none of `thandle`, `key`, or `pos` are provided, it returns the type of the item currently on top of the stack.
    - Passing only `pos` without `thandle` is invalid and returns `FLU_TNONE`.
    - **Side Effect**: This function calls `aot_table_push` (or `flu_getglobal` / `flu_type` directly), which means the queried Lua value is left on top of the stack.

- **`aot_table_first(L, thandle) result(exists)` (Function):**
  Initiates iteration over a table `thandle`. It pushes `nil` onto the stack (as the initial key for `lua_next`) and then calls `flu_next`.
    - If the table has at least one element, the first key-value pair is pushed onto the stack, and the function returns `true`.
    - If the table is empty, it returns `false`.
    - This is intended to be used in conjunction with subsequent `flu_next(L, thandle)` calls to iterate through all table elements.

- **`aot_table_length(L, thandle) result(length)` (Function):**
  Calculates and returns the number of elements in the table `thandle`.
    - **Note:** This function iterates through all key-value pairs in the table using `aot_table_first` and `flu_next`, counting them. This means it counts all elements, not just the sequence part like Lua's `#` operator. For very large, sparse tables, this could be slow.

## Important Variables/Constants

- **`L (type(flu_State))`**: The Lua state handle, passed to all procedures.
- **`thandle (integer)`**: An integer representing the stack index of an opened Lua table. A value of `0` often indicates an invalid or unopened table.
- **`key (character(len=*))`**: String name used to access table elements.
- **`pos (integer)`**: Integer index used to access table elements.
- **Lua Type Constants (e.g., `FLU_TNONE`, `FLU_TNIL`, `FLU_TTABLE` from `flu_binding`):** These integer constants are used by `aot_type_of` and internally by `aot_table_push` to represent Lua data types.

## Usage Examples

**1. Opening a table, pushing an element, getting its type, and closing:**

```fortran
program test_table_ops_basic
  use aot_table_ops_module
  use flu_binding
  implicit none

  type(flu_State) :: L
  integer :: config_table_h, element_type

  L = fluL_newstate()
  call fluL_openlibs(L)

  ! Assume Lua script: config = { settingA = "hello", settingB = 123 }
  call fluL_dostring(L, "config = { settingA = 'hello', settingB = 123 }")

  ! Open the global table 'config'
  call aot_table_open(L, thandle=config_table_h, key="config")
  if (config_table_h == 0) then
    write(*,*) "Failed to open 'config' table."
    call flu_close(L)
    stop
  end if
  write(*,*) "'config' table opened at stack index: ", config_table_h

  ! Push the element 'settingA' from 'config' table onto the stack
  call aot_table_push(L, config_table_h, key="settingA", toptype=element_type)
  write(*,*) "Pushed 'settingA', its Lua type code: ", element_type
  if (element_type == FLU_TSTRING) then
    ! (Value is now on stack, could be retrieved using flu_tolstring)
    write(*,*) "'settingA' is a string."
  end if
  call flu_pop(L, 1) ! Pop 'settingA'

  ! Get type of 'settingB' (also pushes it to stack)
  element_type = aot_type_of(L, config_table_h, key="settingB")
  write(*,*) "Type of 'settingB': ", element_type
  if (element_type == FLU_TNUMBER) then
    write(*,*) "'settingB' is a number."
  end if
  call flu_pop(L, 1) ! Pop 'settingB'

  ! Close the 'config' table
  call aot_table_close(L, config_table_h)
  write(*,*) "'config' table closed. Stack top is now: ", flu_gettop(L)

  call flu_close(L)
end program test_table_ops_basic
```

**2. Iterating through a table:**

```fortran
program test_table_iteration
  use aot_table_ops_module
  use flu_binding
  implicit none

  type(flu_State) :: L
  integer :: my_table_h, key_type, val_type, len
  logical :: has_elements

  L = fluL_newstate()
  call fluL_openlibs(L)
  call fluL_dostring(L, "my_data = { name = 'Test', value = 100, active = true }")

  call aot_table_open(L, thandle=my_table_h, key="my_data")
  if (my_table_h == 0) stop "Table my_data not found"

  len = aot_table_length(L, my_table_h)
  write(*, '(A, I0)') "Calculated table length: ", len

  has_elements = aot_table_first(L, my_table_h) ! Pushes first key-value pair
  if (.not. has_elements) then
    write(*,*) "Table is empty."
  else
    write(*,*) "Iterating through table:"
    do while (has_elements)
      ! Stack: ..., table, key, value
      key_type = flu_type(L, -2) ! Type of the key
      val_type = flu_type(L, -1) ! Type of the value
      write(*, '(A, I0, A, I0)') "  Key type: ", key_type, ", Value type: ", val_type
      ! (Here you could retrieve key and value using flu_to* functions)

      call flu_pop(L, 1) ! Pop value, leave key for next iteration
      has_elements = flu_next(L, my_table_h) ! Pops key, pushes next key-value or nil
    end do
  end if

  call aot_table_close(L, my_table_h)
  call flu_close(L)
end program test_table_iteration
```

## Dependencies and Interactions

- **`flu_binding`**: This is the most fundamental dependency. All procedures in `aot_table_ops_module` make extensive use of `flu_` prefixed functions from `flu_binding` to interact directly with the Lua C API for operations like stack manipulation (`flu_pop`, `flu_gettop`, `flu_settop`), type checking (`flu_isTable`, `flu_type`), getting Lua values (`flu_getfield`, `flu_getglobal`, `flu_getTable`), table creation (`flu_createtable`), and iteration (`flu_next`, `flu_pushnil`).
- **`flu_kinds_module`**: While this module is `use`d, its kind parameters (`double_k`, `single_k`, `long_k`) are not directly part of the procedure signatures or explicit internal logic within `aot_table_ops_module` itself. Their inclusion is likely for consistency or for modules that depend on `aot_table_ops_module`.
- **`aot_table_module`**: This higher-level module directly `use`s `aot_table_ops_module` and re-exports many of its key procedures (`aot_table_open`, `aot_table_close`, `aot_table_top`, `aot_table_length`, `aot_table_first`, `aot_table_push`, `aot_type_of`). `aot_table_module` then builds upon these general operations to provide more specialized, type-safe interfaces for getting and setting various Fortran data types in Lua tables.

`aot_table_ops_module` provides the essential, type-agnostic toolkit for performing basic manipulations and queries on Lua tables from Fortran. It forms a crucial layer for more abstract table interaction modules.
