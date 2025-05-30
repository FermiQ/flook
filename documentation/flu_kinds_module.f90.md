# aotus/LuaFortran/flu_kinds_module.f90

## Overview

This module provides global definitions for Fortran kind parameters for various real and integer data types. By using the intrinsic `selected_real_kind` and `selected_int_kind` functions, it aims to establish consistent numerical precision and range for these types across different compilers and computing platforms. This helps in writing portable and numerically stable Fortran code.

## Key Components

This module does not define any functions, subroutines, or derived types. Its primary components are public parameter constants that define kind values.

## Important Variables/Constants

The module defines the following public integer parameters, which serve as kind selectors for data type declarations:

- **`quad_k = selected_real_kind(33)`**:
  Defines a kind for real numbers with at least 33 decimal digits of precision (quadruple precision).

- **`double_k = selected_real_kind(15)`**:
  Defines a kind for real numbers with at least 15 decimal digits of precision (typically corresponding to standard double-precision floating-point numbers, e.g., 8-byte reals).

- **`single_k = selected_real_kind(6)`**:
  Defines a kind for real numbers with at least 6 decimal digits of precision (typically corresponding to standard single-precision floating-point numbers, e.g., 4-byte reals).

- **`int_k = selected_int_kind(6)`**:
  Defines a kind for integers capable of representing values with at least 6 decimal digits (i.e., up to +/- 10^6 - 1 or greater).

- **`long_k = selected_int_kind(15)`**:
  Defines a kind for integers capable of representing values with at least 15 decimal digits (i.e., up to +/- 10^15 - 1 or greater, often corresponding to 8-byte integers).

## Usage Examples

These kind parameters are used when declaring variables to specify their precision or range.

```fortran
module my_calculations
  use flu_kinds_module
  implicit none

  real(kind=single_k) :: temperature_measurement
  real(kind=double_k) :: precise_calculation_result
  real(kind=quad_k)   :: high_precision_constant

  integer(kind=int_k) :: loop_counter
  integer(kind=long_k) :: large_identifier

  ! Example usage
  temperature_measurement = 37.5_single_k
  precise_calculation_result = 1.23456789012345_double_k
  high_precision_constant = 3.14159265358979323846264338327950288_quad_k

  loop_counter = 1000_int_k
  large_identifier = 987654321098765_long_k

  ! ... further computations ...

end module my_calculations
```

## Dependencies and Interactions

- **Fortran Intrinsic Functions:**
    - `selected_real_kind(p, r)`: Used to determine the kind parameter for real types based on desired decimal precision `p` and exponent range `r` (range is not specified in this module, so default is used).
    - `selected_int_kind(r)`: Used to determine the kind parameter for integer types based on the desired decimal range `r` (number of digits).

- **Other Project Modules:**
  This module is expected to be used by other Fortran modules within the `aotus/LuaFortran` project (and potentially other related projects) to ensure consistent type definitions for numerical stability and portability. For example, `flu_binding.f90` uses `int_k` and `long_k`.

It has no dependencies on other custom modules within the project.
