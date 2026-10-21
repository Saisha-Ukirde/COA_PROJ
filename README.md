# Pipeline Execution Simulation

An interactive, single file simulation of a five stage instruction pipeline. It lets you step through the clock one cycle at a time, compare pipelined execution against one-instruction-at-a-time execution, and see how data hazards create stalls.

**Author:** Saisha Ukirde
**Roll number:** 2125UIDF1102
**Submitted to:** Prof. Ankita

---

## Table of contents

1. Overview
2. Quick start
3. Features
4. Interface guide
5. The pipeline stages
6. The sample program
7. How the simulator schedules instructions
8. Worked example
9. Results at a glance
10. Project structure
11. Design notes
12. Assumptions and limitations
13. Ideas for extension
14. Browser support

---

## 1. Overview

A processor that finishes one instruction completely before starting the next leaves most of its hardware idle at any moment. Pipelining fixes this by splitting instruction execution into stages and letting several instructions occupy different stages at the same time, much like an assembly line.

This project visualises that idea. It runs a short six instruction program through a classic five stage pipeline and shows:

- which instruction sits in which stage on the current clock cycle,
- a cycle by cycle grid of the whole execution,
- how many cycles, stalls and cycles per instruction (CPI) the run has used so far.

Everything lives in one file, `index.html`, with no libraries, no build step and no network access required.

## 2. Quick start

1. Download or copy `index.html` to any folder.
2. Double click it, or drag it into a browser window.
3. Press **Run** to watch the pipeline execute, or **Step** to advance one clock cycle at a time.

No installation, server or internet connection is needed.

## 3. Features

- **Step or run:** advance manually one cycle per click, or let the clock run automatically at slow, normal or fast speed.
- **Two execution modes:** pipelined, and one instruction at a time, so the speedup can be compared directly.
- **Data hazard toggle:** turn read-after-write hazards on or off to see stalls appear and disappear.
- **Live stage view:** five boxes show the instruction currently in fetch, decode, execute, memory and write back.
- **Cycle grid:** a table with one row per instruction and one column per clock cycle, filled in as time advances. The current cycle is outlined.
- **Running statistics:** clock cycle, instructions finished, stall cycles and CPI.
- **Plain language note:** a line under the grid explains what is happening in the current mode.
- **Keyboard accessible:** all controls can be reached with the Tab key and have visible focus outlines.
- **Responsive layout:** works on a laptop screen and on a phone, with the grid scrolling sideways when it is too wide.

## 4. Interface guide

| Control | What it does |
|---|---|
| Run | Starts the automatic clock. The button changes to Pause while running. If the program has finished, Run starts it again from cycle 0. |
| Step | Pauses the automatic clock if it is running and advances exactly one cycle. |
| Reset | Stops the clock and returns to cycle 0 with an empty grid. |
| Pipelined | Instructions overlap in the pipeline. |
| One at a time | Each instruction runs all five stages before the next one begins. |
| Data hazards, no forwarding | When checked, an instruction that needs a register value not yet written back must wait. Only available in pipelined mode. |
| Speed | Sets the time between automatic clock ticks: Slow (1.1 s), Normal (0.65 s) or Fast (0.25 s). |

Changing the mode or the hazard checkbox resets the simulation so that the grid always matches the selected settings.

### Reading the statistics

| Value | Meaning |
|---|---|
| Clock cycle | Number of cycles elapsed so far. |
| Instructions finished | Instructions that have completed write back. |
| Stall cycles | Number of cycles in which at least one instruction was held waiting for a hazard to clear. |
| Cycles per instruction | Elapsed cycles divided by finished instructions. Lower is better. An ideal pipeline approaches 1. |

### Reading the grid

- Coloured cells show the stage an instruction occupies in that cycle. Each instruction keeps its own muted colour so a row is easy to follow.
- Grey cells marked **stall** show cycles where the instruction was fetched but could not move on to decode.
- The outlined column is the cycle currently being executed.

## 5. The pipeline stages

| Stage | Name | Work done |
|---|---|---|
| IF | Fetch | Read the instruction from instruction memory. |
| ID | Decode | Decode the instruction and read its source registers. |
| EX | Execute | Perform the arithmetic or compute the memory address. |
| MEM | Memory access | Load from or store to data memory. |
| WB | Write back | Write the result into the destination register. |

Every instruction passes through all five stages and each stage takes exactly one clock cycle. The simulator does not distinguish between instruction types in the later stages, so a register instruction still spends one cycle in MEM, as in the textbook model.

## 6. The sample program

The program is fixed and chosen so that it contains both dependent and independent instructions.

| # | Instruction | Writes | Reads |
|---|---|---|---|
| 1 | `LW  R1, 0(R2)` | R1 | R2 |
| 2 | `ADD R3, R1, R4` | R3 | R1, R4 |
| 3 | `SUB R5, R3, R6` | R5 | R3, R6 |
| 4 | `AND R7, R8, R9` | R7 | R8, R9 |
| 5 | `OR  R10, R7, R11` | R10 | R7, R11 |
| 6 | `SW  R5, 4(R2)` | none | R5, R2 |

Dependencies that cause stalls when hazards are enabled:

- Instruction 2 needs R1 from instruction 1.
- Instruction 3 needs R3 from instruction 2.
- Instruction 5 needs R7 from instruction 4.

Instruction 4 is independent. Instruction 6 reads R5, but instruction 3 has already written it back by the time instruction 6 reaches decode, so it does not stall.

## 7. How the simulator schedules instructions

The schedule for the whole program is computed up front, and the clock simply reveals it one cycle at a time. For instruction `i`, the simulator finds the cycle in which it enters each stage.

### One at a time mode

Instruction `i` starts fetch at cycle `5 * i` and then occupies the next four stages in consecutive cycles. There are no stalls and no overlap.

### Pipelined mode

1. **Fetch:** the first instruction is fetched in cycle 0. Each later instruction is fetched in the same cycle that the previous instruction moves from fetch into decode.
2. **Decode:** an instruction can enter decode no earlier than one cycle after its own fetch, and no earlier than one cycle after the previous instruction entered decode (decode handles one instruction per cycle).
3. **Hazard check (if enabled):** if the instruction reads a register written by an earlier instruction, decode must wait until the cycle in which that earlier instruction performs write back.
4. **Remaining stages:** once an instruction enters decode, it flows through EX, MEM and WB in consecutive cycles without further delay.

While an instruction waits for decode, it stays in the fetch stage. The live stage view labels it as held, and the grid shows grey stall cells. Because the next instruction cannot be fetched until this one leaves fetch, a stall delays everything behind it, which matches how an in-order pipeline behaves.

### Register file timing

The register file is assumed to write in the first half of a cycle and read in the second half. A dependent instruction can therefore decode in the same cycle that its producer writes back, which is why a stalled instruction resumes exactly on the producer's write back cycle.

## 8. Worked example

Pipelined, hazards on. Columns are clock cycles (1 based, as displayed in the grid). `st` marks a stall.

```
        1   2   3   4   5   6   7   8   9   10  11  12  13  14  15  16
LW      IF  ID  EX  MEM WB
ADD         IF  st  st  ID  EX  MEM WB
SUB                     IF  st  st  ID  EX  MEM WB
AND                                 IF  ID  EX  MEM WB
OR                                      IF  st  st  ID  EX  MEM WB
SW                                                  IF  ID  EX  MEM WB
```

What to notice:

- `ADD` is fetched in cycle 2 but cannot decode until cycle 5, the cycle in which `LW` writes R1. It loses two cycles.
- `SUB` waits the same way for `ADD` to write R3.
- `AND` has no dependency, so it moves straight through, but it is still delayed because it was fetched late.
- `OR` waits for `AND` to write R7.
- The six stall cycles (3, 4, 6, 7, 10 and 11) are what the Stall cycles counter reports at the end of the run.

## 9. Results at a glance

| Mode | Total cycles | Stall cycles | CPI at finish |
|---|---|---|---|
| One at a time | 30 | 0 | 5.00 |
| Pipelined, hazards off | 10 | 0 | 1.67 |
| Pipelined, hazards on | 16 | 6 | 2.67 |

Speedup over one at a time:

- Pipelined without hazards: 30 / 10 = 3.0 times faster.
- Pipelined with hazards: 30 / 16 = about 1.9 times faster.

For six instructions the ideal pipeline needs `5 + (6 - 1) = 10` cycles, which is the general formula `k + (n - 1)` for `k` stages and `n` instructions. As `n` grows, CPI approaches 1. The short program here keeps the fill time of the pipeline visible, which is why the CPI is above 1 even in the ideal case.

## 10. Project structure

```
.
├── index.html    the complete simulation (markup, styles and script)
└── README.md     this document
```

Inside `index.html`:

| Part | Purpose |
|---|---|
| `<style>` block | Layout, colours and typography. Colours are defined as CSS variables at the top. |
| `<main>` markup | Controls, stage boxes, statistics, grid container and footer. |
| `PROG` array | The six instructions, each with a destination register and a list of source registers. |
| `build()` | Computes the full schedule for the selected mode and hazard setting. |
| `what(i, t)` | Returns what instruction `i` is doing in cycle `t`: a stage name, `stall`, or nothing. |
| `render()` | Redraws the stage boxes, statistics, grid and note for the current cycle. |
| `tick()`, `start()`, `stop()`, `reset()` | Clock control. |

## 11. Design notes

- **Minimal interface:** one neutral background, one accent colour, no gradients, shadows, icons or animation. The interface shows only what is needed to follow the simulation.
- **Colour has a job:** each instruction has its own muted tint so its path through the stages can be followed across the grid. Grey is reserved for stalls and idle stages.
- **Plain wording:** labels use ordinary words (Run, Step, Reset) and sentence case.
- **Monospace only for code:** instruction text uses a monospace font so register names line up. Everything else uses the system font.
- **Accessibility:** controls are real buttons and form fields, focus outlines are visible, and the current mode is exposed to assistive technology through `aria-pressed`.
- **Easy to restyle:** change the CSS variables in `:root` to adjust the palette without touching anything else.

## 12. Assumptions and limitations

This is a teaching model, so it keeps to the standard textbook simplifications.

- Five stages, one cycle each, no variable latency.
- In-order, single issue: one instruction enters the pipeline per cycle at most.
- Only read-after-write (RAW) data hazards are modelled.
- No forwarding or bypassing. With forwarding, most of the stalls shown here would disappear, and only the load followed immediately by a dependent instruction would still need a one cycle stall.
- No control hazards: the program has no branches or jumps.
- No structural hazards: instruction memory and data memory are assumed to be separate, so fetch and memory access never conflict.
- The program is fixed in the source and cannot be edited from the interface.

## 13. Ideas for extension

- Add a forwarding toggle and compare stall counts with and without it.
- Let the user type or paste their own instruction sequence.
- Add branch instructions with a flush penalty to show control hazards.
- Add a register table that updates at write back.
- Export the finished grid as an image or CSV file.
- Add a stage by stage latency setting to model a slower memory stage.

## 14. Browser support

Works in any current version of Chrome, Edge, Firefox or Safari. It uses plain HTML, CSS and JavaScript with no external dependencies.
