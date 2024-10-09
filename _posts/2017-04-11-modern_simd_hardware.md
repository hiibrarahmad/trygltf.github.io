---
layout: post
title: "Modern SIMD Programming in the SSE Era"
---
For a very long time now with every new gaming console generation came a new set of hardware. New hardware meant new constraints which meant we had to revisit old optimization tricks, rules, and popular beliefs.

With the latest generation, both leading consoles ([Xbox One](https://en.wikipedia.org/wiki/Xbox_One) and [PlayStation 4](https://en.wikipedia.org/wiki/PlayStation_4)) have opted for [x64](https://en.wikipedia.org/wiki/X86-64) and [AMD CPUs](https://en.wikipedia.org/wiki/Advanced_Micro_Devices). These are indeed significantly different from the old [PowerPC CPUs](https://en.wikipedia.org/wiki/PowerPC) used by [Xbox 360](https://en.wikipedia.org/wiki/Xbox_360) and [PlayStation 3](https://en.wikipedia.org/wiki/PlayStation_3).

I spent a significant amount of time last year optimizing cloth simulation code that had been written in mind for the previous generation and with a few tweaks I was able to get decent gains on the latest hardware. Unfortunately that code was proprietary and as such I cannot use it to show the changes and lessons I learned. Instead, I will take the decent and well respected [DirectX Math](https://github.com/Microsoft/DirectXMath) public library and highlight as well as optimize specific examples.

Due to the large amount of code, charts, and information required to properly explain everything, this topic will be split into several parts:

*  [Hardware highlights](#hardware_highlights)
*  [Matrix multiplication]({% post_url 2017-04-13-modern_simd_matrix_multiplication %})
*  And much more to come…

# <a name="hardware_highlights"></a>Hardware highlights

## The PowerPC era

In the Xbox 360 and PlayStation 3 generation, both CPUs were quite different, in particular the PlayStation 3 CPU was a significant departure from modern trends at the time. We will focus on it today only because it has been more publicly documented than its counterpart and the lessons we’ll learn from it applied to both consoles.

The PlayStation 3 console sported a fancy [Cell microprocessor](https://en.wikipedia.org/wiki/Cell_(microprocessor)). We will not focus on the multi-threading aspects in this post series and instead take a deeper look at the execution of a single thread at the assembly level. One important characteristic of PowerPC chips is that they are based on the [RISC](https://en.wikipedia.org/wiki/Reduced_instruction_set_computing) model. RISC hardware is generally known to have more user addressable registers than [CISC](https://en.wikipedia.org/wiki/Complex_instruction_set_computing), the variant used by Intel and AMD for modern x64 CPUs. In fact, PowerPC CPUs have 32 general purpose registers, 32 floating point registers, and 32 [AltiVec](https://en.wikipedia.org/wiki/AltiVec) SIMD registers (note that the Cell SPEs had 128 registers!). Both consoles also had cache lines that were 128 bytes wide and their CPUs did not support [out-of-order execution](https://en.wikipedia.org/wiki/Out-of-order_execution).

This gave rise to two common techniques to optimize code at the assembly level that we will revisit in this series.

First, the large amount of registers meant that we could leverage them easily to mitigate the fact that loading values from memory was slow and could not be easily hidden due to the in-order nature of the execution. This lead to register packing: a technique where values are loaded first in bulk as part of a vector value and specific components are extracted on demand.

Second, because the register sets for floating point and SIMD math were different, moving a value from one set to another involved storing the value in memory (generally the stack) and reloading it. This led to what is commonly known as [load-hit-store](http://assemblyrequired.crashworks.org/load-hit-stores-and-the-__restrict-keyword/) stalls. To mitigate this, whenever SIMD math was mixed with scalar math it was generally best to treat simple scalar floating point math as if it were SIMD: the same scalar was replicated to every SIMD lane.

It is also worth noting that the calling convention used on those consoles favours passing many arguments by register, including vector arguments. There are also many other peculiarities that affected how code should best be written for the hardware such as avoiding to stress the poor branch prediction but we will not cover these at this time.

## The x64 SSE era

The modern Xbox One and PlayStation 4 consoles opted for x64 AMD CPUs. These are state of the art processors with out-of-order execution, powerful branch prediction, and a large set of instructions from SSE and AVX. These CPUs depart from the previous generation significantly in the number of registers they have: 16 general purpose registers and 16 SIMD registers. The cache lines are also standard in size: 64 bytes wide.

The greatly reduced number of registers means that x64 code is much more prone to register pressure issues. Whenever the number of registers needed goes above 16, registers will spill on the stack, degrading performance. In practice, it isn’t a huge issue in large part because x64 supports instructions that directly work from memory, avoiding the need for a separate load instructions (unlike PowerPC CPUs). For example, in the expression `C = add(A, B)`, **A** and **B** can both be residing in registers or **B** could optionally be a memory address. Internally the CPU has much more than 16 registers and thanks to register renaming and compound CISC instructions, our `add` instruction will end up performing a load for us behind the scenes. Leveraging this can be very important in hot code as we will see in this series.

Another important characteristic of x64 is that scalar floating point math uses the SSE registers. This means that unlike with PowerPC, converting a floating point value into a SIMD value is very cheap (it could be free if you hand write assembly but compilers will generally generate a shuffle instruction regardless of whether or not you need all components). On the other hand, converting a SIMD value into a floating point value is entirely free and no instruction is generated by modern compilers. This is an important factor to take into account and as we will see later in this series, it can be leveraged to great effect.

# Up next

In the next post we will take a deep look at how 4x4 matrix multiplication is performed in DirectX Math and how we can speed it up in various ways.

