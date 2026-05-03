# MSP430FR5969 CPU Cycle Measurement Guide

Measuring the CPU cycle consumption for functions and programs at the assembly level on the MSP430 family launchpad


## Objective

This document explains how to:

1. Create a simple MSP430FR5969 project in CCS
2. Write a C program with:

   * `add()` function
   * `sub()` function
   * `main()` function
3. Generate executable output for MSP430FR5969
4. Measure CPU cycles for:

   * individual functions
   * whole program execution
5. Visualize values in CCS IDE
6. Use breakpoints and debugging tools correctly

---

# 1. Hardware and Software Setup

## Hardware

* MSP430FR5969 LaunchPad

## Software

* Code Composer Studio (CCS) v20.5.0
* TI LLVM/Clang Compiler
* Windows 10/11

---

# 2. Important Concept

The MSP430FR5969 does NOT contain:

* ARM DWT cycle counter
* dedicated hardware CPU cycle counter register

Therefore, CPU cycles are measured using:

## Timer_A

Timer_A runs from SMCLK.

If:

* CPU clock = 1 MHz
* SMCLK = 1 MHz

then:

```text
1 timer tick ≈ 1 CPU cycle
```

This provides approximate runtime cycle measurement.

---

# 3. Create CCS Project

## Step 1

Open CCS.

## Step 2

Go to:

```text
File → New → CCS Project
```

## Step 3

Use the following configuration:

| Option       | Value         |
| ------------ | ------------- |
| Device       | MSP430FR5969  |
| Project Type | Empty Project |
| Language     | C             |
| Compiler     | TI LLVM/Clang |
| Debug Probe  | XDS110        |

Click:

```text
Finish
```

---

# 4. Disable Compiler Optimization

This is VERY IMPORTANT.

If optimization is enabled:

* functions may be inlined
* measurements become invalid
* compiler may remove code

## Steps

Go to:

```text
Project Properties
→ Build
→ MSP430 Compiler
→ Optimization
```

Set:

```text
-O0
```

Apply and close.

---

# 5. Full Program

Create a file named:

```text
main.c
```

Paste the following code.

```c
#include <msp430.h>
#include <stdint.h>

/*----------------------------------------------------------
 * Global Variables
 *---------------------------------------------------------*/

volatile uint16_t add_cycles = 0;
volatile uint16_t sub_cycles = 0;
volatile uint16_t program_cycles = 0;

volatile int add_result = 0;
volatile int sub_result = 0;

/*----------------------------------------------------------
 * Prevent compiler inlining
 *---------------------------------------------------------*/

__attribute__((noinline))
int add(int a, int b)
{
    return a + b;
}

__attribute__((noinline))
int sub(int a, int b)
{
    return a - b;
}

/*----------------------------------------------------------
 * Timer Initialization
 * SMCLK source
 * Continuous mode
 *---------------------------------------------------------*/

void timer_init(void)
{
    TA0CTL = TASSEL__SMCLK | MC__CONTINUOUS | TACLR;
}

/*----------------------------------------------------------
 * Main
 *---------------------------------------------------------*/

int main(void)
{
    uint16_t start, end;

    uint16_t prog_start, prog_end;

    /* Stop Watchdog */
    WDTCTL = WDTPW | WDTHOLD;

    /* Initialize Timer */
    timer_init();

    /*======================================================
     * START WHOLE PROGRAM MEASUREMENT
     *=====================================================*/

    prog_start = TA0R;

    /*======================================================
     * Measure add()
     *=====================================================*/

    start = TA0R;

    add_result = add(10, 5);

    end = TA0R;

    add_cycles = end - start;

    /*======================================================
     * Measure sub()
     *=====================================================*/

    start = TA0R;

    sub_result = sub(10, 5);

    end = TA0R;

    sub_cycles = end - start;

    /*======================================================
     * END WHOLE PROGRAM MEASUREMENT
     *=====================================================*/

    prog_end = TA0R;

    program_cycles = prog_end - prog_start;

    /* Infinite loop for debugger observation */

    while(1)
    {

    }
}
```

---

# 6. Build Project

## Build

Use:

```text
Ctrl + B
```

OR:

```text
Project → Build Project
```

This generates the MSP430 executable.

Example:

```text
Debug/project_name.out
```

---

# 7. Flash and Debug

Connect:

* MSP430FR5969 LaunchPad

Then:

```text
Run → Debug
```

CCS will:

1. compile code
2. flash MSP430
3. enter debug mode

---

# 8. Add Watch Variables

Open:

```text
View → Expressions
```

Add the following variables:

```text
add_cycles
sub_cycles
program_cycles

add_result
sub_result
```

---

# 9. Add Breakpoint

## Recommended Breakpoint Location

Place a breakpoint at:

```c
while(1)
```

Reason:

* all measurements are completed
* all variables contain final values
* safe observation point

---

# 10. Run Program

Press:

```text
F8
```

The program stops at:

```c
while(1)
```

Now inspect the variables.

---

# 11. Expected Output

Example results:

| Variable       | Example Value |
| -------------- | ------------- |
| add_result     | 15            |
| sub_result     | 5             |
| add_cycles     | 21            |
| sub_cycles     | 19            |
| program_cycles | 45            |

---

# 12. Important Observation

## Why is:

```text
program_cycles > add_cycles + sub_cycles
```

?

Because:

`program_cycles` includes:

* function call overhead
* return overhead
* variable assignments
* timer reads
* memory accesses
* main() execution overhead
* branch/control instructions

Whereas:

* `add_cycles` measures only add() measurement region
* `sub_cycles` measures only sub() measurement region

---

# 13. What is Included in Function Cycle Count?

The measurement:

```c
start = TA0R;

add_result = add(10, 5);

end = TA0R;
```

includes:

1. timer register read
2. function argument setup
3. CALL instruction
4. stack operations
5. arithmetic operation
6. RET instruction
7. result storage
8. timer register read

This is realistic runtime measurement.

---

# 14. Visualizing Results in CCS

## A. Expressions Window

Shows:

* cycle counts
* results
* global variables

---

## B. Variables Window

Open:

```text
View → Variables
```

Shows:

* local variables
* global variables
* stack variables

---

## C. Registers Window

Open:

```text
View → Registers
```

Important registers:

| Register | Purpose         |
| -------- | --------------- |
| PC       | Program Counter |
| SP       | Stack Pointer   |
| SR       | Status Register |

These will later be important for checkpointing research.

---

## D. Disassembly Window

Open:

```text
View → Disassembly
```

You should see instructions like:

```asm
CALL #add
CALL #sub
```

along with:

* MOV
* ADD
* SUB
* RET

This becomes important for:

* instruction-level cycle analysis
* LLVM-based program analysis
* energy modeling

---

## E. Memory Browser

Open:

```text
View → Memory Browser
```

Useful for:

* inspecting FRAM
* observing stack memory
* checkpointing experiments later

---

# 15. Important Debugging Notes

## DO NOT

Measure cycles by:

* stepping instruction-by-instruction
* using single-step repeatedly

Reason:

* debugger halts CPU
* timing becomes invalid

Correct method:

1. run program freely
2. stop AFTER measurements complete

---

# 16. Why **attribute**((noinline)) is Important

Without:

```c
__attribute__((noinline))
```

compiler may:

* inline functions
* remove CALL instructions
* invalidate function-level measurements

Therefore always use:

```c
__attribute__((noinline))
```

for cycle-analysis experiments.

---

# 17. Research Importance of This Experiment

This experiment establishes:

| Concept                   | Research Relevance         |
| ------------------------- | -------------------------- |
| Function execution timing | Program profiling          |
| Timer-based measurement   | Runtime cycle estimation   |
| CCS visualization         | Runtime debugging          |
| Disassembly analysis      | Instruction-level analysis |
| Registers observation     | Checkpointing              |
| Memory observation        | FRAM persistence           |

This becomes the foundation for:

* intermittent execution
* energy modeling
* LLVM instrumentation
* checkpoint placement
* task-based execution

---

# 18. Next Planned Steps

Future experiments will include:

1. Exact MSP430 instruction-level cycle analysis
2. Assembly inspection
3. Static cycle estimation
4. LLVM IR generation
5. Basic-block cycle counting
6. Energy estimation
7. GPIO timing measurement
8. Intermittent execution experiments
9. FRAM checkpointing
10. Recovery runtime implementation

---

# 19. Common Problems and Fixes

## Problem 1

### Functions disappear in assembly

### Cause

Compiler optimization enabled.

### Fix

Set:

```text
-O0
```

and use:

```c
__attribute__((noinline))
```

---

## Problem 2

### Variables not visible in watch window

### Cause

Optimization removes variables.

### Fix

Declare variables as:

```c
volatile
```

---

## Problem 3

### Cycle counts inconsistent

### Cause

Debugger stepping or optimization.

### Fix

* run freely
* use breakpoint only after measurements
* keep optimization disabled

---

# 20. Summary

This experiment successfully demonstrates:

1. MSP430 executable generation
2. Function-level cycle counting
3. Whole-program cycle counting
4. CCS debugging workflow
5. Runtime observation techniques
6. Foundation for intermittent computing research

The next step will focus on:

* exact instruction cycle analysis
* assembly-level timing
* LLVM-based analysis
* energy estimation infrastructure

