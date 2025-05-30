# aotus/source/aot_err_module.f90

## Overview

The `aot_err_module` provides centralized error handling capabilities for the AOTUS (Adaptive Optics Toolbox for Usage with Scripts) library, focusing on errors that arise from interactions with Lua scripts. It defines a set of error codes (`aoterr_*`) to categorize specific issues encountered when trying to access or use Lua values. The core of the module is the `aot_err_handler` subroutine, designed to capture and process error messages from the Lua environment.

## Key Components

- **Error Code Parameters:**
    - **`aoterr_Fatal = 0`**: Integer parameter. The comment suggests it indicates a fatal error if this "bit is set", though its value is 0.
    - **`aoterr_NonExistent = 1`**: Integer parameter. Indicates that a requested value does not exist within the Lua script.
    - **`aoterr_WrongType = 2`**: Integer parameter. Indicates that a requested value exists in the Lua script but is of an incorrect data type for the operation being attempted.
    *(The comments suggest these are bits in a bitmask, but their values 0, 1, 2 don't align with typical power-of-2 bitmasking. They seem to function more as distinct error identifiers or categories based on their current values.)*

- **`aot_err_handler(L, err, msg, ErrString, ErrCode)` (Subroutine):**
    - This is the central error handling routine. It is intended to be called after operations involving the Lua API (typically via `flu_binding` functions) that return an error code.
    - **Arguments:**
        - `L (type(flu_State))`: The Lua state handle.
        - `err (integer, intent(in))`: The error code returned by the Lua operation.
        - `msg (character(len=*), intent(in))`: An additional custom message to be prepended to any Lua error message, especially if the program stops.
        - `ErrString (character(len=*), intent(out), optional)`: If present, receives the error message retrieved from the Lua stack.
        - `ErrCode (integer, intent(out), optional)`: If present, receives the original Lua error code (`err`).
    - **Behavior:**
        - If `err` is non-zero (indicating an error):
            - It retrieves the detailed error message from the Lua stack using `flu_tolstring`.
            - If `ErrString` is provided, the Lua error message is copied into it.
            - If `ErrCode` is provided, the input `err` is copied into it.
            - If neither `ErrString` nor `ErrCode` is present (i.e., the caller doesn't want to handle the error explicitly), the subroutine writes the combined `msg` and Lua error message to standard output and then `STOP`s program execution.
            - If `ErrString` or `ErrCode` (or both) are present, program execution continues, and the caller is responsible for handling the error using the returned information.

## Important Variables/Constants

- **`aoterr_Fatal`, `aoterr_NonExistent`, `aoterr_WrongType`**: These parameters are used to classify errors. While their usage as a bitmask is mentioned in comments, their current values suggest they might be used as distinct codes.
- **`err` (in `aot_err_handler`)**: The input Lua error code that triggers the handler's logic.
- **`ErrString`, `ErrCode` (in `aot_err_handler`)**: Optional output arguments that allow the calling routine to receive error details without halting the program. Their presence dictates whether the `aot_err_handler` will stop execution or pass control back.

## Usage Examples

The `aot_err_handler` is designed to be used immediately after calls to Lua functions (via `flu_binding`) that can produce an error status.

```fortran
module data_loader
  use flu_binding
  use aot_err_module
  implicit none

  subroutine load_data_from_lua(L_state, script_file)
    type(flu_State), intent(inout) :: L_state
    character(len=*), intent(in) :: script_file

    integer :: lua_error_status
    character(len=512) :: error_message_str
    integer :: error_code_val

    ! Attempt to load a Lua file
    lua_error_status = fluL_loadfile(L_state, script_file)

    ! Call aot_err_handler to process potential errors
    ! Example 1: Stop program if an error occurred
    ! call aot_err_handler(L_state, lua_error_status, "Error loading script: " // trim(script_file))

    ! Example 2: Handle error explicitly if it occurred
    call aot_err_handler(L_state, lua_error_status, "Error loading script: " // trim(script_file), &
                         ErrString=error_message_str, ErrCode=error_code_val)

    if (error_code_val /= 0) then ! LUA_OK is typically 0
      write(*, '(A, I0, A, A)') "Failed to load Lua script. Error Code: ", error_code_val, &
                                 " Message: ", trim(error_message_str)
      ! Perform cleanup or other error recovery actions
      return
    end if

    ! ... proceed with other Lua operations, calling aot_err_handler after each ...

    ! Example of calling a Lua function
    lua_error_status = flu_pcall(L_state, 0, 0, 0) ! (0 args, 0 results, no error function)
    call aot_err_handler(L_state, lua_error_status, "Error executing Lua script's main chunk", &
                         ErrString=error_message_str, ErrCode=error_code_val)
    if (error_code_val /= 0) then
        write(*,*) "Lua pcall failed: ", trim(error_message_str)
        ! Handle error
    end if

  end subroutine load_data_from_lua

end module data_loader
```

## Dependencies and Interactions

- **`flu_binding`:**
    - The `aot_err_handler` takes a `type(flu_State)` argument, which is defined in `flu_binding`.
    - It uses `flu_tolstring` (from `flu_binding`) to retrieve the error message string from the Lua stack.
- **Calling AOTUS Routines:** Any part of the AOTUS system that interacts with Lua and uses functions that return an error code (typically from `flu_binding`, like `fluL_loadfile`, `flu_pcall`, etc.) should use `aot_err_handler` to manage potential Lua errors.
- **Lua C API:** Indirectly, through `flu_binding`, this module depends on the behavior of the Lua C API, especially how error messages are pushed onto the Lua stack.

This module standardizes error reporting and handling for Lua-related operations within the AOTUS framework, providing a choice between automatic program termination on error or allowing the calling code to manage errors gracefully.
