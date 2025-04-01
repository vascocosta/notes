# Array vs Slice vs String vs Buffer

1.  **Array (`[N]T`)**
    *   **Definition:** A fixed-size collection of elements of the same type `T`, where the size `N` is known at **compile time**.
    *   **Size:** The size `N` is part of the array's type. `[5]u8` is a different type than `[6]u8`.
    *   **Memory:** Typically allocated directly on the stack (for local variables) or in the data segment (for globals/constants). The array *owns* its memory.
    *   **Mutability:** Can be mutable (`var`) or immutable (`const`).
    *   **Example:**
        ```zig
        var numbers: [5]i32 = .{ 10, 20, 30, 40, 50 }; // Array of 5 i32s
        const name_bytes: [4]u8 = ."Zig!";       // Array of 4 u8s (from string literal)
        numbers[0] = 11; // Modifiable if 'var'
        ```
    *   **Key Takeaway:** Fixed size known at compile time. Size is part of the type. Owns its data.

2.  **Slice (`[]T` or `[]const T`)**
    *   **Definition:** A dynamically-sized *view* or *reference* into a contiguous sequence of elements of type `T`. It consists internally of a **pointer** to the first element and a **length** (`len`).
    *   **Size:** The length is determined at **runtime** and is stored alongside the pointer. The length is *not* part of the slice's type (`[]i32` is always the same type, regardless of its runtime length).
    *   **Memory:** A slice *borrows* memory. It **does not own** the underlying data it points to. The data must be kept alive elsewhere (e.g., in an array, allocated on the heap).
    *   **Mutability:**
        *   `[]T`: A mutable slice. Allows modifying the elements it points to.
        *   `[]const T`: An immutable (read-only) slice. Does *not* allow modifying the elements through this slice, even if the underlying memory is mutable.
    *   **Creation:** Often created by "slicing" an array or another slice, or from memory returned by an allocator.
    *   **Example:**
        ```zig
        var numbers: [5]i32 = .{ 10, 20, 30, 40, 50 };

        // Create a mutable slice pointing to elements 1, 2, 3
        var middle_slice: []i32 = numbers[1..4]; // len = 3
        middle_slice[0] = 21; // Modifies numbers[1]

        // Create an immutable slice pointing to the whole array
        const all_slice: []const i32 = &numbers; // Equivalent to numbers[0..numbers.len]

        // Cannot modify through a const slice:
        // all_slice[0] = 1; // Compile Error!

        std.debug.print("numbers: {any}\n", .{numbers}); // Output: numbers: { 10, 21, 30, 40, 50 }
        ```
    *   **Key Takeaway:** Runtime length. Pointer + Length. Borrows memory (doesn't own it). `[]T` is mutable view, `[]const T` is immutable view.

3.  **"String" (`[]const u8`)**
    *   **Definition:** This is not a distinct *type* in Zig. It's a **convention**. A "string" in Zig is typically represented as an immutable slice of bytes (`[]const u8`).
    *   **Why `u8`?** This naturally handles ASCII. For UTF-8 (Zig's standard encoding), operations often work directly on the underlying bytes. The standard library provides functions for UTF-8 validation and manipulation.
    *   **Memory:** Like any slice, it borrows memory. String literals (`"hello"`) produce `[]const u8` slices that point to statically allocated, read-only memory.
    *   **Mutability:** By convention, strings are immutable slices (`[]const u8`). If you need a modifiable sequence of bytes (e.g., to build a string), you'd typically use a mutable slice (`[]u8`), often backed by an array or allocated memory (which we often call a "buffer").
    *   **Example:**
        ```zig
        const greeting: []const u8 = "Hello, Zig!"; // String literal becomes []const u8
        var message_buffer: [20]u8 = undefined;
        const message_slice: []u8 = &message_buffer; // Mutable slice for building

        // Copy greeting into the buffer
        @memcpy(message_slice[0..greeting.len], greeting);

        // Treat a portion as an immutable string
        const part: []const u8 = message_slice[0..5]; // "Hello"
        std.debug.print("Part: {s}\n", .{part});
        ```
    *   **Key Takeaway:** A conventional type alias: `[]const u8`. It's just a specific kind of immutable slice, used for representing text.

4.  **Buffer**
    *   **Definition:** This is not a specific Zig *type* but rather a **usage pattern** or **role**. A buffer is typically a region of **mutable memory**, usually represented by a `[]u8` slice (or sometimes directly an array like `[N]u8`), intended to be **written into**.
    *   **Purpose:** Used for tasks like:
        *   Receiving data from I/O operations (reading from files, network).
        *   Building strings piece by piece (e.g., with `std.fmt.bufPrint`).
        *   Temporary storage during data manipulation.
    *   **Memory:** The memory backing the buffer slice can come from various places:
        *   A stack-allocated array (`var my_buf_arr: [100]u8 = undefined; var buf: []u8 = &my_buf_arr;`).
        *   Heap-allocated memory (`var buf: []u8 = try allocator.alloc(u8, 100); defer allocator.free(buf);`).
    *   **Mutability:** Almost always mutable (`[]u8` or `var [N]u8`), because its primary purpose is to be written to.
    *   **Example (using `std.fmt.bufPrint`):**
        ```zig
        const std = @import("std");

        var buffer_array: [50]u8 = undefined; // Array providing the memory
        var buffer_slice: []u8 = &buffer_array; // Mutable slice acting as the buffer

        const name = "Alice";
        const age = 30;

        // Use the buffer_slice to format a string into it
        const formatted_slice = try std.fmt.bufPrint(buffer_slice, "User: {s}, Age: {d}", .{ name, age });
        // formatted_slice is now a []const u8 view of the part of buffer_slice that was written

        std.debug.print("Formatted: {s}\n", .{formatted_slice}); // Output: Formatted: User: Alice, Age: 30
        ```
    *   **Key Takeaway:** A *role* for mutable memory (often `[]u8` or `[N]u8`) intended for writing data into. Not a distinct language type itself.

**Summary Table:**

| Feature        | Array (`[N]T`)                | Slice (`[]T`, `[]const T`)     | "String" (`[]const u8`)       | Buffer (`[]u8`, `[N]u8`)       |
| :------------- | :---------------------------- | :----------------------------- | :---------------------------- | :----------------------------- |
| **Kind**       | Language Type                 | Language Type                  | Convention (is a slice)       | Usage Pattern (uses slice/array) |
| **Size**       | Fixed, compile-time (`N`)   | Dynamic, runtime (`len`)     | Dynamic, runtime (`len`)    | Dynamic (`len`) or Fixed (`N`) |
| **Size in Type** | Yes (`N` is part of type)     | No                             | No                            | Only if array (`[N]u8`)        |
| **Memory**     | Owns                          | Borrows                        | Borrows                       | Owns (if array) or Borrows (if slice view) |
| **Mutability** | `var` or `const`              | `[]T` (mutable view), `[]const T` (immutable view) | Conventionally `[]const u8` (immutable view) | Usually mutable (`[]u8`, `var [N]u8`) |
| **Typical Use**| Fixed collections, data structs | Views into data, function args | Text representation         | I/O, string building, temp storage |
