# LEGv8 Local Sort

A sorting program written entirely in **LEGv8 (ARMv8-style) assembly**. It fills an array with pseudorandom 64-bit integers, sorts it using a local-swap algorithm built from small procedures, and then verifies the result with a recursive binary search.

Course project for Iowa State University, Com S 3210 (Computer Architecture), Spring 2026.

## What it does

1. **Generates data:** fills a 512-element array of 8-byte integers using a linear congruential generator (seed 42).
2. **Sorts it:** repeatedly scans the array, swapping any element that is out of order with its neighbor, until a full pass makes no swaps.
3. **Verifies it:** binary-searches for every element (all must be found) and for a few values that should be absent (none may be found).

## Procedures

| Procedure | Purpose |
|---|---|
| `fill_array` | Fills the array with `seed = (seed * 1103515245 + 12345) & 0x7FFFFFFF`, one value per element |
| `swap` | Swaps two array elements by index |
| `is_locally_sorted` | Checks whether the element at index `i` is in order with its immediate neighbors |
| `local_sort` | Drives the sort: calls `is_locally_sorted` and `swap` until no swaps occur in a pass |
| `binary_search` | Recursive binary search; returns 1 if found, 0 if not |
| `main` | Sets up the array, runs the sort, and runs all the search checks |

## Design notes

- **Multiplication with shifts and adds.** The multiplier 1103515245 is built from 15 shifted copies of the seed added together, and the increment 12345 is added as repeated immediates. The mask `0x7FFFFFFF` is built with shifts and ORs because immediates are only 12 bits.
- **No nested loops.** The sort is one loop that calls small helpers. `is_locally_sorted` decides whether index `i` needs attention, and `local_sort` decides which neighbor to swap with.
- **Recursive binary search.** The midpoint is `n / 2`. Searching the right half advances the base pointer to `a + (mid + 1) * 8` and shrinks the length to `n - mid - 1`. Searching the left half keeps the base and uses length `mid`.
- **Calling conventions.**
  - Arguments are passed in `X0`-`X2`, and results are returned in `X0`.
  - Procedures that call others (`local_sort`, `binary_search`) push a stack frame and save the link register plus the callee-saved registers they use (`X19`-`X22`).
  - Leaf procedures (`fill_array`, `swap`, `is_locally_sorted`) use only temporary registers and need no frame.
  - `local_sort` keeps its "swapped" flag in a saved register (`X22`) so it survives the calls to `swap` and `is_locally_sorted`.

## Memory layout

The array starts at address 0 and uses 512 x 8 = 4096 bytes, which fits exactly in the emulator's default 4 KiB of main memory. The stack is separate and accessed through `SP`.

## Running it

The program runs on the course-provided LEGv8 emulator (Linux only):

```
legv8emul assignment1.legv8asm
```

The emulator ends with a core dump, and an arrow (`-->`) marks the `HALT` that was reached.

- Arrow on the `HALT` right after the last search check: every check passed.
- Arrow on the `HALT` under `fail:`: some search gave the wrong answer.

To test quickly, change `ADDI X20, XZR, #512` in `main` to a small value such as `#8`.

## Results

With seed 42 and n = 512, all 512 elements are found after sorting and all the absent-value checks (`a[0] - 1`, `a[0] + 1`, `a[n-1] + 1`) come back not found. The run executes about 10.07 million instructions.

Note: `a[0] + 1` can legitimately appear in the array for other seeds, so that check is only guaranteed to pass for seeds where the value is absent.

## What I learned

- Stack frames in assembly are manual: reserve space, store each saved register at its own offset, restore in the same layout, then release the space. Reusing one offset for every save silently corrupts the return address.
- Keeping state across calls requires a callee-saved register. Temporaries like `X9`-`X15` can be overwritten by any procedure you call.
- Debugging without a debugger meant testing each procedure on a small `n` first and reading core dumps.

## Author

Fatima Qureshi, Iowa State University
