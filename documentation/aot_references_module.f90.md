# aotus/source/aot_references_module.f90

## Overview

The `aot_references_module` provides a robust mechanism for Fortran code to create and use stable references to Lua objects (such as tables or functions). It leverages Lua's built-in reference system, which allows Lua values to be stored in a special, globally accessible table called the "registry" and retrieved later using an integer key.

This approach is particularly useful when Fortran needs to hold onto a Lua object across several operations or function calls without the need to repeatedly navigate potentially complex Lua table structures by keys or indices. This module is the modern, recommended replacement for the now-obsolete `aot_path_module`. The concepts are further explained in "Programming in Lua" (PiL) section 27.3.2.

## Key Components

- **`aot_reference_for(L, thandle, key, pos) result(ref)` (Function):**
  This function creates an integer reference to a Lua object.
    - **Arguments:**
        - `L (type(flu_State))`: The active Lua state.
        - `thandle (integer, intent(in), optional)`: If provided, this is expected to be a handle (e.g., stack index) to a Lua table from which the object will be retrieved.
        - `key (character(len=*), intent(in), optional)`: If `thandle` is provided, `key` is the string key of the object within `thandle`. If `thandle` is *not* provided, `key` is treated as the name of a global Lua variable.
        - `pos (integer, intent(in), optional)`: If `thandle` is provided and `key` is not, `pos` is the integer index of the object within `thandle`. This argument is ignored if `key` is provided or if `thandle` is not.
    - **Behavior:**
        1.  The target Lua object is first pushed onto the Lua stack.
            - If `thandle` is present, `aot_table_push` is used to push `thandle[key]` or `thandle[pos]`.
            - If only `key` is present, `flu_getglobal` pushes the global variable `key`.
            - If none of `thandle`, `key`, or `pos` are provided, the object currently on top of the Lua stack is used.
        2.  `fluL_ref(L, LUA_REGISTRYINDEX)` is then called. This pops the object from the stack, stores it in the Lua registry, and returns an integer reference (`ref`).
    - **Returns:**
        - `ref (integer)`: The integer reference (key) to the Lua object now stored in the registry.

- **`aot_reference_to_top(L, ref)` (Subroutine):**
  This subroutine uses a previously obtained integer reference to push the corresponding Lua object back onto the top of the Lua stack.
    - **Arguments:**
        - `L (type(flu_State))`: The active Lua state.
        - `ref (integer)`: An integer reference obtained from a prior call to `aot_reference_for`.
    - **Behavior:**
        - Calls `flu_rawgeti(L, LUA_REGISTRYINDEX, ref)`, which retrieves the Lua object associated with `ref` from the registry and pushes it onto the stack.

## Important Variables/Constants

- **`LUA_REGISTRYINDEX` (from `lua_parameters`):** This integer constant is a pseudo-index that always refers to the Lua registry table. It's fundamental to the operation of `fluL_ref` and `flu_rawgeti`.
- **`ref` (integer):** The integer value returned by `aot_reference_for` acts as a unique key within the Lua registry for the stored object. This same key is then used by `aot_reference_to_top`.

## Usage Examples

```fortran
program test_lua_references
  use flu_binding
  use aot_references_module
  use aot_table_ops_module ! For aot_table_open (conceptual)
  use aot_fun_module       ! For aot_fun_type, aot_fun_do (conceptual)
  use lua_parameters       ! For LUA_GLOBALSINDEX (conceptual, if needed)
  implicit none

  type(flu_State) :: L
  integer :: my_table_ref, my_func_ref
  integer :: table_handle_on_stack
  type(aot_fun_type) :: func_handle_aot

  ! 1. Initialize Lua and load a script (conceptual)
  L = fluL_newstate()
  call fluL_openlibs(L)
  ! Assume a script is loaded:
  ! my_global_table = { setting = 123, run_me = function() print("Function run!") end }
  ! Call fluL_dostring(L, "my_global_table = { setting = 123, run_me = function() print('Function run!') end }")
  ! For a real file, use fluL_loadfile then flu_pcall.

  ! 2. Get a reference to 'my_global_table'
  my_table_ref = aot_reference_for(L, key="my_global_table")
  if (my_table_ref == LUA_REFNIL .or. my_table_ref == LUA_NOREF) then ! Check for valid ref
    write(*,*) "Could not get reference for my_global_table"
    call flu_close(L)
    stop
  end if
  write(*,*) "Got reference for my_global_table: ", my_table_ref

  ! 3. Get a reference to the function 'run_me' inside 'my_global_table'
  ! First, push 'my_global_table' onto the stack using its reference
  call aot_reference_to_top(L, my_table_ref) ! my_global_table is now at stack top (-1)
  table_handle_on_stack = flu_gettop(L)      ! Get its stack index

  ! Now get reference for 'run_me' within the table at table_handle_on_stack
  my_func_ref = aot_reference_for(L, thandle=table_handle_on_stack, key="run_me")
  call flu_pop(L,1) ! Pop my_global_table, we only needed it to get the func ref

  if (my_func_ref == LUA_REFNIL .or. my_func_ref == LUA_NOREF) then
      write(*,*) "Could not get reference for run_me function"
  else
      write(*,*) "Got reference for run_me function: ", my_func_ref
  end if

  ! ... (later in the code, possibly in another subroutine) ...

  ! 4. Use the function reference to call 'run_me'
  if (my_func_ref /= LUA_REFNIL .and. my_func_ref /= LUA_NOREF) then
    call aot_reference_to_top(L, my_func_ref) ! Pushes 'run_me' function to stack top

    ! Use aot_fun_module to handle the function call (conceptual)
    func_handle_aot = aot_fun_top(L) ! aot_fun_top expects function at stack top
    if (func_handle_aot%handle /= 0) then
        call aot_fun_do(L, func_handle_aot, 0) ! 0 results expected
        call aot_fun_close(L, func_handle_aot) ! Clean up
        write(*,*) "'run_me' function called using its reference."
    else
        write(*,*) "Failed to prepare function from reference for calling."
        call flu_pop(L,1) ! Pop the item that wasn't a function
    end if
  end if

  ! 5. Release references (important to avoid memory leaks in Lua)
  !    Lua provides luaL_unref for this. This module doesn't expose it directly,
  !    but it's a critical part of using references.
  !    call fluL_unref(L, LUA_REGISTRYINDEX, my_table_ref)
  !    call fluL_unref(L, LUA_REGISTRYINDEX, my_func_ref)
  !    (Assuming fluL_unref is available in flu_binding or similar)

  ! 6. Close Lua state
  call flu_close(L)

end program test_lua_references
```

## Dependencies and Interactions

- **`flu_binding`**: This is a core dependency.
    - `flu_State`: The Lua state type.
    - `flu_getglobal(L, key)`: Used to get a global variable onto the stack.
    - `fluL_ref(L, LUA_REGISTRYINDEX)`: The Lua C API function (via Fortran interface) that creates the reference in the registry.
    - `flu_rawgeti(L, LUA_REGISTRYINDEX, ref)`: The Lua C API function (via Fortran interface) that retrieves an object from the registry using its reference.
- **`lua_parameters`**:
    - `LUA_REGISTRYINDEX`: Provides the essential constant for the registry table index.
    - (Implicitly, constants like `LUA_REFNIL` or `LUA_NOREF` would also come from here or `flu_binding` for checking validity of references).
- **`aot_table_ops_module`**:
    - `aot_table_push(L, thandle, key, pos)`: Used by `aot_reference_for` to push a specific element from a Lua table onto the stack before it's referenced.
- **Application Code**: Other modules in AOTUS or user applications would use `aot_references_module` to obtain and use these integer references as stable handles to Lua objects, especially when these objects need to be accessed multiple times or passed around in Fortran code.
- **Comparison to `aot_path_module`**: This module provides a more direct and Lua-idiomatic way to "bookmark" Lua objects compared to the string-based path construction of the obsolete `aot_path_module`. Lua references are generally more efficient and less prone to errors from path misconfigurations.
- **Memory Management**: It's important to note that while this module helps create references, Lua's garbage collector will not collect objects as long as they are referenced in the registry. A corresponding `luaL_unref` call (which would need to be exposed via `flu_binding` or a similar low-level interface) is necessary to release these references when they are no longer needed to allow Lua to garbage collect the objects if there are no other references. This module does not provide an explicit "unreference" routine.

This module offers a cleaner and more standard Lua approach for Fortran to maintain persistent handles to specific Lua data structures or functions.
