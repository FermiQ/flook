# aotus/source/aot_fun_module.f90

## Overview

The `aot_fun_module` provides a comprehensive suite of tools for Fortran programs to interact with Lua functions. It establishes a clear workflow for calling Lua functions:

1.  **Open/Access a Lua function**: Using `aot_fun_open` (which can find functions in tables or via direct references). This yields an `aot_fun_type` handle.
2.  **Push Arguments**: Use the `aot_fun_put` interface to supply arguments to the Lua function. It supports scalar real numbers and 1D arrays of reals (which are passed as Lua tables).
3.  **Execute the Function**: Call `aot_fun_do` to invoke the Lua function with the previously supplied arguments.
4.  **Retrieve Results**: After execution, results left on the Lua stack by the Lua function can be retrieved using routines from other modules (e.g., `aot_top_get_val` from `aot_top_module`, though not directly part of this module, it's mentioned as the next step in the intended workflow).
5.  **Repeat (Optional)**: Steps 2-4 can be repeated if the same function needs to be evaluated multiple times with different arguments.
6.  **Close the Function**: Finally, `aot_fun_close` should be called to clean up the Lua stack and invalidate the function handle.

The module also includes error handling for function execution and provides a way to get a unique string identifier for a function.

## Key Components

- **`aot_fun_type`**:
  Re-exported from `aot_fun_declaration_module`. This derived type is used to store information about an opened Lua function, including its handle (stack position or reference), argument count, and a unique ID.

- **`aot_fun_open` (Public Interface):**
    - **`aot_fun_table(L, parent, fun, key, pos)`**: Retrieves a Lua function located within a Lua table. The function can be identified by its string `key` or integer `pos` within the table. The `parent` argument specifies the table's location on the stack.
    - **`aot_fun_ref(L, fun, ref)`**: Retrieves a Lua function using a previously obtained Lua reference (`ref`).

- **`aot_fun_put` (Public Interface):**
  Used to push arguments onto the Lua stack in preparation for calling an opened function.
    - **`aot_fun_put_top(L, fun)`**: Takes the value currently on top of the Lua stack and uses it as the next argument for the function `fun`.
    - **`aot_fun_put_double(L, fun, arg)`**: Pushes a double-precision real scalar `arg`.
    - **`aot_fun_put_single(L, fun, arg)`**: Pushes a single-precision real scalar `arg` (converted to double for Lua).
    - **`aot_fun_put_double_v(L, fun, arg)`**: Pushes a 1D array of double-precision reals `arg`. The array is converted into a new Lua table, which then becomes the argument.
    - **`aot_fun_put_single_v(L, fun, arg)`**: Pushes a 1D array of single-precision reals `arg`. The array is converted to double precision and then into a new Lua table argument.
    *(The module also `use`s `aot_quadruple_fun_module` and `aot_extdouble_fun_module`, suggesting that the `aot_fun_put` interface is likely extended by procedures from these modules to support higher precision real types.)*

- **`aot_fun_top(L) result(fun)` (Function):**
  Takes the item currently on top of the Lua stack. If it's a Lua function, it initializes an `aot_fun_type` handle (`fun`) for it, including storing its stack position and a pointer-based ID. It also pushes a copy of the function onto the stack to ensure the reference is maintained across calls.

- **`aot_fun_close(L, fun)` (Subroutine):**
  Closes an opened Lua function represented by `fun`. This typically involves cleaning up the Lua stack by removing the function and its arguments/results above its initial handle position. Resets the `fun` variable.

- **`aot_fun_do(L, fun, nresults, ErrCode, ErrString)` (Subroutine):**
  Executes the Lua function `fun` that has been prepared with arguments.
    - `nresults`: Specifies the number of results expected from the Lua function.
    - `ErrCode`, `ErrString`: Optional arguments to retrieve Lua error codes and messages. If not provided, the subroutine will stop execution on error, printing a message.
  After execution, `fun%arg_count` is set to -1, indicating the function needs resetting (re-pushing arguments) if it's to be called again.

- **`aot_fun_id(fun) result(id)` (Function):**
  Returns a 32-character string representation of the unique ID (`fun%id`) associated with the opened function.

## Important Variables/Constants

- The `aot_fun_type` derived type is central to managing Lua functions. Its components (`handle`, `arg_count`, `id`) track the state of an opened function.
- The `arg_count` component of `aot_fun_type` is dynamically updated:
    - Incremented by `aot_fun_put_*` routines.
    - Set to `-1` by `aot_fun_do` to indicate that the function has been called and its arguments consumed/popped. This state triggers a reset logic in `aot_fun_put_*` routines if arguments are pushed again for a subsequent call.

## Usage Examples

The intended workflow is described in the module's introductory comments:

```fortran
module main_program
  use flu_binding         ! For flu_State, fluL_newstate, fluL_openlibs, flu_close
  use aot_fun_module      ! The module being documented
  use aot_top_module      ! For aot_top_get_val (hypothetical retrieval)
  use aot_err_module      ! For error constants (if needed, though aot_fun_do handles errors)
  implicit none

  type(flu_State) :: L
  type(aot_fun_type) :: my_lua_func
  integer :: err_c, num_results_expected
  character(len=256) :: err_s
  real(kind=double_k) :: arg1, result_val

  ! 1. Initialize Lua State
  L = fluL_newstate()
  call fluL_openlibs(L)
  ! ... (Load Lua script file, e.g., using fluL_loadfile and aot_err_handler) ...
  ! Assume script defines: function myLuaAdder(a, b) return a + b end

  ! 2. Open the Lua function (e.g., from global scope, treated as a table)
  !    To open a global function, parent can be LUA_GLOBALSINDEX (from lua_parameters)
  !    or use a specific table handle if the function is in a table.
  !    For this example, let's assume it's a global function 'myLuaAdder'.
  !    (Note: Opening globals directly might need flu_getglobal then aot_fun_top,
  !     or a utility not shown here. aot_fun_table is for functions *within* tables.)

  !    Alternative: if 'myLuaAdder' is global, push it first:
  err_c = flu_getglobal(L, "myLuaAdder")
  if (err_c == LUA_TFUNCTION) then ! LUA_TFUNCTION from lua_parameters
      my_lua_func = aot_fun_top(L)
  else
      write(*,*) "Could not find or open function myLuaAdder"
      call flu_close(L)
      stop
  end if

  ! 3. Put arguments
  arg1 = 10.5_double_k
  call aot_fun_put(L, my_lua_func, arg1) ! Uses aot_fun_put_double

  call flu_pushNumber(L, 20.2_double_k) ! Push second arg directly
  call aot_fun_put(L, my_lua_func)      ! Uses aot_fun_put_top

  ! 4. Execute the function
  num_results_expected = 1
  call aot_fun_do(L, my_lua_func, num_results_expected, ErrCode=err_c, ErrString=err_s)

  if (err_c == 0) then ! LUA_OK
    ! 5. Retrieve results (example, actual retrieval uses aot_top_get_val)
    !    Results are on the stack. If num_results_expected = 1, one item is there.
    !    call aot_top_get_val(L, result_val) ! (from aot_top_module)
    if (flu_isNumber(L, -1)) then
        result_val = flu_tonumber(L, -1)
        write(*,*) "Lua function 'myLuaAdder' returned:", result_val
    else
        write(*,*) "Lua function did not return a number."
    end if
    call flu_pop(L, num_results_expected) ! Clean up results from stack
  else
    write(*,*) "Error during aot_fun_do: ", trim(err_s)
  end if

  ! 6. Close the function
  call aot_fun_close(L, my_lua_func)

  ! 7. Clean up Lua state
  call flu_close(L)

end module main_program
```

## Dependencies and Interactions

- **`flu_binding`**: This is a primary dependency. `aot_fun_module` uses numerous `flu_` prefixed functions for fundamental Lua C API interactions (stack management, type checking, function calls, etc.).
- **`flu_kinds_module`**: Provides `double_k` and `single_k` for defining real variable kinds.
- **`aot_fun_declaration_module`**: Supplies the definition of `aot_fun_type`.
- **`aot_table_module`**:
    - `aot_table_push`: Used by `aot_fun_table` to place a table or its element (the function) on top of the stack.
    - `aot_table_from_1Darray`: Used by `aot_fun_put_double_v` and `aot_fun_put_single_v` to convert Fortran arrays into Lua tables for argument passing.
- **`aot_top_module`**:
    - The `aot_err_handler` (likely re-exported or used via `aot_top_module`) is employed by `aot_fun_do` to manage errors from `flu_pcall`.
    - The module comments suggest `aot_top_get_val` (from `aot_top_module`) as the means to retrieve results after `aot_fun_do`.
- **`aot_references_module`**:
    - `aot_reference_to_top`: Used by `aot_fun_ref` to push a Lua object (the function) onto the stack using its reference.
- **`aot_quadruple_fun_module` & `aot_extdouble_fun_module`**: These modules are `use`d, indicating that `aot_fun_module` likely extends its `aot_fun_put` interface to support quadruple precision and other extended double precision real types through procedures defined in these specialized modules.

This module acts as a sophisticated layer for Fortran-Lua function calls, abstracting many direct Lua stack manipulations into a more structured, stateful (via `aot_fun_type`) approach.
