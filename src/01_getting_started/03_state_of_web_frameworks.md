# The State of Rust web frameworks


While some great web frameworks are out there, there is room for another point of view.
Some of them are either not very flexible, too complicated, pushed compile time errors to runtime by using
the type `std::any::Any` or all of the above.

With continuation of the Rust way, we would have to extend the Rust guarantees:

- Outstanding runtime performance for concurrent workloads.
- Flexible, testable and composable software.
- Minimizing runtime errors, as much as possible.
- Simplicity.