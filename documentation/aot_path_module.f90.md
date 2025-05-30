# aotus/source/aot_path_module.f90

**WARNING: This module is considered obsolete. The `aot_references_module` should be used for functionally similar purposes. This module might be removed in future versions of AOTUS.**

## Overview

The `aot_path_module` was designed to provide a mechanism for tracking and accessing Lua entities (specifically tables and functions) that are nested within other Lua tables. It allowed for the construction of a "path" to an entity, which could then be used to open and interact with that entity from Fortran. The path itself is represented as a linked list of nodes, where each node specifies an element in the Lua table hierarchy (identified by a string key or an integer position).

Despite its capabilities, this module is **obsolete**, and developers are directed to use the `aot_references_module` for more current and supported methods of accessing Lua entities.

## Key Components

- **`aot_path_node_type` (Private Derived Type):**
  The fundamental building block of a path. Each node stores:
    - `NodeType (character(len=16))`: Type of the Lua entity at this node (e.g., "table", "function").
    - `ID_kind (character(len=16))`: How the node is identified within its parent table ("key" or "position").
    - `key (character(len=80))`: The string key if `ID_kind` is "key".
    - `pos (integer)`: The integer position if `ID_kind` is "position".
    - `child (type(aot_path_node_type), pointer)`: Pointer to the next node in the path.

- **`aot_path_type` (Public Derived Type):**
  Represents the complete path to a Lua entity. It contains:
    - `LuaFilename (character(len=256))`: Name of the Lua script file where the path is relevant.
    - `rootHandle (integer)`: A handle to the top-most table that was opened when traversing the path.
    - `GlobalNode (type(aot_path_node_type), pointer)`: Pointer to the first node in the linked list representing the path.
    - `head (type(aot_path_node_type), pointer)`: Pointer to the last node currently in the path (where new nodes are added).

- **Path Manipulation Subroutines:**
    - **`aot_init_path(me, Filename)`**: Initializes an `aot_path_type` object, optionally setting the Lua filename and clearing any existing nodes.
    - **`aot_fin_path(me)`**: Finalizes an `aot_path_type`, deallocating all nodes in its linked list.
    - **`aot_path_addNode(me, NodeType, pos, key)`**: Appends a new node to the end of the path, specifying its type and identifier (position or key).
    - **`aot_path_delNode(me, isEmpty)`**: Removes the last node from the path. Optionally returns whether the path becomes empty.

- **Assignment Operator:**
    - **`interface assignment(=)` (for `module procedure aot_path_copy`)**: Overloads the assignment operator (`=`) for `aot_path_type` to perform a deep copy of the path, including all its nodes.

- **Path Traversal and Usage Interfaces:**
    - **`aot_path_open` (Public Interface):**
        - `aot_path_open_fun(me, conf, fun, openLua)`: Traverses the path `me` within the Lua state `conf` and opens the Lua function at the end of the path.
        - `aot_path_open_table(me, conf, thandle, openLua)`: Traverses the path `me` and opens the Lua table at the end of the path.
        *(The `openLua` argument allows these routines to optionally open the Lua script file specified in `me%LuaFilename`.)*
    - **`aot_path_close` (Public Interface):**
        - `aot_path_close_fun(me, conf, fun, closeLua)`: Closes a function opened via `aot_path_open_fun` and any intermediate tables.
        - `aot_path_close_table(me, conf, closeLua)`: Closes tables opened via `aot_path_open_table`.
        *(The `closeLua` argument allows these routines to optionally close the Lua script.)*

- **Utility Subroutines:**
    - **`aot_path_toString(path, pathAsString)`**: Converts the `aot_path_type` into a human-readable string (e.g., `myTable.subKey[1].targetFunction`).
    - **`aot_path_dump(path, outputUnit)`**: Prints a detailed representation of the path and its nodes to the specified Fortran `outputUnit`, for debugging.

## Important Variables/Constants

- **`NodeType` values**: Strings like "table" or "function" are used to define what kind of Lua entity a path node refers to.
- **`ID_kind` values**: Strings "key" or "position" determine how a node is looked up within its parent Lua table.

## Usage Examples

**Note: The following example is for illustrative purposes only, as this module is obsolete.**

```fortran
program test_aot_path_obsolete
  use aot_path_module
  use flu_binding, only: flu_State
  use aotus_module, only: open_config_file, close_config_file_by_state => close_config
  use aot_fun_module, only: aot_fun_type ! Only for type, not for direct open/close here
  implicit none

  type(aot_path_type) :: my_path
  type(flu_State) :: L
  type(aot_fun_type) :: target_function_handle
  integer :: final_table_handle
  character(len=200) :: path_str

  ! Initialize Lua state and load a script (conceptual)
  ! call open_config_file(L, "my_script.lua")
  ! Assume my_script.lua contains:
  ! config_table = {
  !   settings = {
  !     my_func = function(a, b) return a + b end
  !   }
  ! }

  ! 1. Initialize the path
  call aot_init_path(my_path, Filename="my_script.lua")

  ! 2. Build the path
  ! Path to config_table.settings.my_func
  call aot_path_addNode(my_path, NodeType="table", key="config_table")
  call aot_path_addNode(my_path, NodeType="table", key="settings")
  call aot_path_addNode(my_path, NodeType="function", key="my_func")

  ! 3. Convert path to string (optional)
  call aot_path_toString(my_path, path_str)
  ! write(*,*) "Path to open: ", trim(path_str) ! Expected: config_table.settings.my_func

  ! 4. Open the function using the path (opens my_script.lua implicitly if openLua=.true.)
  ! call aot_path_open(my_path, L, target_function_handle, openLua=.true.)

  ! if (target_function_handle%handle /= 0) then
  !   write(*,*) "Function opened successfully via path."
  !   ! ... (use target_function_handle with aot_fun_put, aot_fun_do from aot_fun_module) ...

  !   ! 5. Close the function and path
  !   call aot_path_close(my_path, L, target_function_handle, closeLua=.true.)
  ! else
  !   write(*,*) "Failed to open function via path."
  !   ! if openLua was true, L might need closing if it was opened:
  !   ! call close_config_file_by_state(L)
  ! end if

  ! 6. Finalize path object
  call aot_fin_path(my_path)

  ! Make sure L is closed if it was opened and not closed by aot_path_close
  ! (This part of example is simplified as L's state depends on openLua logic)

  write(*,*) "Reminder: aot_path_module is obsolete. Use aot_references_module."

end program test_aot_path_obsolete
```

## Dependencies and Interactions

- **`flu_binding`**: Provides the `flu_State` derived type, representing the Lua state.
- **`aotus_module`**: The `open_config_file` and `close_config` procedures are used by `aot_path_open_*` and `aot_path_close_*` respectively, when the `openLua` or `closeLua` optional arguments are set to true, to manage the lifecycle of the Lua script file itself.
- **`aot_table_module`**: The `aot_table_open` and `aot_table_close` procedures are used internally by `aot_path_open_table` and `aot_path_close_table` (and by extension, the function-opening variants) to navigate through the nested Lua tables defined in the path.
- **`aot_fun_module`**: The `aot_fun_type` derived type and the `aot_fun_open` / `aot_fun_close` procedures from this module are used when the terminal node of a path is a function.
- **`aot_references_module`**: This is the **recommended replacement** for `aot_path_module`. Users should migrate to `aot_references_module` for robust and supported ways to access Lua entities.

The primary interaction pattern of this module involves building a path descriptor and then using it to command the AOTUS library to navigate Lua tables to reach a specific, possibly deeply nested, Lua table or function. Due to its obsolete status, further development or reliance on this module is discouraged.
