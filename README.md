# General purpose CPU powered by littlemen

The idea of this particular design has only occured to me once the [2026 ICFP contest](https://icfpcontest2026.com) was over, so this isn't a contest writeup per se. This article is more like a followup research on how much of desktop CPU features can be achieved with those little guys. The implementation is at a proof-of-concept level, lacking certain flexibility and commercial-grade tools.

# Design philosophy

One of the things learnt from building handmade solvers to the contest problems is that it's very difficult for a littleman to *request* data. If a littleman needs to ask something from his neighbours, then sending a specific request to the right neighbour would require moving closer to his respective outgoing pipe first, and that wastes precious ticks, if possible at all. And after that, waiting for the response takes time too, which isn't good for the performance either.

A much more efficient way for littlemen to process data is to *receive* all required data without asking, along with the instruction to process it. Instead of actively asking for data, most littlemen should be mere reactors waiting for the event to come, and then performing their job once the expected event arrives.

Another thing learnt from the contest is that pipelining the operations is also very important for the performance. Eeach littleman should be assigned a task as simple as handling one unit of data or performing one math operation. Everything more complex than that'd better be decomposed into a sequence of simplier functions chained in monadic manner. This way each littleman doesn't have to wait for the entire chain to complete; he can start a new operation as soon as he finishes his sub-task.

Decomposition also makes it possible to perform some operations in parallel, which also speeds the things up.

All that said, the guiding philosophy for the CPU design would be:

* make everything pipeline-oriented;
* decompose every operation into a chain of simple steps;
* let each step work asynchronously;
* let the steps work in parallel as much as possible;
* strive for high throughput;

# Top to bottom design

I'll start with the big picture and then gradually refine it, going into details where necessary.

Basically, the goal of each computer system is to be able to process user data under the control of user program. It means that at the top level the system should comprise the unit that processes the data and the unit that feeds that processor with commands:

![top_level.svg](images/top_level.svg?fileId=83510#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

Let's start with the processing part and get back to the program source later. For now, just assume that it provides a constant stream of operation codes somehow.

## Mixer

Design principles stated above lead to the idea that user data should come with the instruction somehow. Each instruction should carry all required data, and each processing room should be kind of pure function that acts on it. Well, almost pure because a room may have some state that persists between the events -- without that memory cells and registers wouldn't be possible.

So, user data should be represented as a stream. It should meet the stream of opcodes and mix with it to produce the stream of instructions. Those instructions should go to some device(s) that will ultimately execute them:

![mixer.svg](images/mixer.svg?fileId=83619#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

Upon executing the instruction, a matching processing device will likely produce some result, and it seems natural to loop that result back to the data input pipe:

![mixer_loop.svg](images/mixer_loop.svg?fileId=83801#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

That's about all there is to the processing core.

Mixer is the main active component that makes the rest of the CPU tick. It works like a watch escapement: it releases instructions from incoming opcode queue one by one, at regular intervals. And it lets data produced by one instruction to be consumed by another instruction.

Then, not every operation requires an argument. For example, reading a value from I-room doesn't need any argument at all.

And some other operations may require more than one argument, for example adding two values. Well, to keep the things as simple as possible, multi-argument operations will be decomposed, as each littleman reactor will hardly be capable of handling multiple units of data anyway.

The bottom line here is that the mixer must inspect the opcode to tell whether the current operation needs an argument from the data pipe. Hence, opcodes should be designed with that in mind.

## Instruction dispatching

The next challenge is decoding the opcodes and dispatching each instruction to the right executor. Any kind of demultiplexer would take O(log N) ticks at best and could grow rather big in size, if the set of instructions is large and sparse.

This is where parallel computations come handy. Let each instruction from the mixer enter a distribution bus that will broadcast it to all processing devices at once. And let each device decide if it should act upon each incoming instruction.

A bus is just a long and narrow room with a littleman who retransmits every value received from any incoming pipe to all outgoing pipes. Distribution bus has one incoming pipe and many outgoing pipes, it's used to forward instruction to all devices. Collector bus has lots of incoming pipes and one outgoing pipe, it's used to collect results from many processing devices to a single data pipe.

A simple layout could be like this:

![device_line.svg](images/device_line.svg?fileId=83988#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

A more intricate layout that allows scaling in both dimensions is:

![device_grid.svg](images/device_grid.svg?fileId=83989#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

## Processing device

Each processing device consists of a gate and a worker. The gate inspects each incoming instruction and tries to match its opcode against the factory-configured mask of this particular gate. If the opcode matches, then the instruction is passed on to the worker. The gate also strips the opcode from the instruction, so that the worker receives instructions in their pure form, and only those instructions that were addressed to his respective device.

![device.svg](images/device.svg?fileId=84013#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

The rest is up to the worker. He may process the instruction as he sees fit: store some state in his second hand or in the backpack, share some data with neighbour workers, produce one or more data units as his output.

The only thing to care is that instructions keep coming no matter what. The worker must complete his task by the time the next instruction could arrive from the gate.

If the worker missed his deadline enough times in a row, then the gate man will get blocked at his send operation, stalling the gate, which in turn would clog and stall its respective branch of the big instruction bus. But other independent branches, that aren't clogged, will keep running, and that will result in unsolicited out-of-order execution. And that would ruin the order of data units in the incoming pipe of the mixer, and then stalled instructions would get wrong arguments, and... Well, that's bad enough already.

The bottom line is that each worker absolutely must complete his task on time, all possible branches considered. Longer task should be decomposed to a chain of shorter tasks, each one of them fast enough to meet the time limit. If that cannot't be done, then the gate should be followed by some load-balancing round-robin dispatcher that would pass each matching instruction to one of K equivalent workers. Or, as the last resort, the compiler could be tweaked to keep an eye on this particular opcode and avoid issuing it too often.

## Instruction format

Now that the contract of the gate is specified, it's possible to design the format of opcodes and instructions.

The fact that each gate sees every instruction implies that each gate must be able to skip any instruction that's addressed to someone else. That's easy if all instructions have the same length and format. As mentioned above, the most simple format looks good enough:

![instruction.svg](images/instruction.svg?fileId=84014#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

Each instruction consists of one opcode followed by one argument.

The mixer must stick to this format no matter whether the current opcode actually needs an argument or not. If the current opcode requires an argument, then the mixer should pick the next data unit from data pipe and produce instruction `{opcode, data}`. If the current opcode doesn't need an argument, then the mixer should produce instruction `{opcode, 0}` merely to keep the format uniform.

The opcode should also have a fixed layout so that the mixer can easily tell if it should pick an argument from the incoming data pipe.

![opcode_addr.svg](images/opcode_addr.svg?fileId=84018#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

The intuition behind such format is that from the mixer point of view, there are only two instructions:

* query something from the device at the given fixed address;
* send a piece of data to the device at the given fixed address.

The rest of opcode is just the address of the device that should handle the given query/send operation.

Each device is free to produce a data result in responce to either instruction.

Also, each device is free to trade a few lower bits of its address for an immediate argument, if it needs one:

![opcode_imm.svg](images/opcode_imm.svg?fileId=84019#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

Such trade is controlled by the mask which is hardcoded in each gate.

### Index registers, memory mapping

Data processing opcodes may contain only fixed addresses, thus providing access only to fixed bus devices, including memory cells.

However a general purpose CPU must offer some indirect addressing modes, that would let the program access a device (or a memory cell) whose address is calculated at runtime. Littlemen CPU offers indirect addressing in the form `[pointer_register + fixed_offset]` by performing runtime address translation.

A special device called memory mapper can be attached to the common instruction bus, and that device will intercept all instructions addressed to some specific memory region (1Kb pages in the reference implementation) and map these instructions to other addresses by adding the value of `pointer_register` to the fixed address of a matched instruction.

![page_mapping.svg](images/page_mapping.svg?fileId=84020#mimetype=image%2Fsvg%2Bxml&hasPreview=false)Several such devices could be connected in parallel, offering N independent index registers to access memory cells in the same secondary instruction bus.

To address a memory cell indirectly (that is, to dereference a pointer in a higher level language), user program must calculate the desired address, store that address to the pointer register of any memory mapper, and then access fixed address `0` within the page controlled by that mapper. For example, if the program decided to use the first mapped page that spans addresses 0x0400 -- 0x07FF, then accessing address `0x0400` will actually map to `[pointer_register + 0]`, accessing address `0x0401` will map to `[pointer_register + 1]`, and so on.

One memory mapper could be reserved for `this` or `self` pointer of object-oriented languages. Accessing fixed address with offset `X` in the page controlled by that mapper would access member with offset `X` in the object. Pretty easy.

## Response lag

Each instruction goes a long way from the mixer to the worker, and then the result produced by the worker travels nearly as long way back to the mixer. Thus far we've seen the following steps:

* instruction escapes the mixer and enters the distribution bus;
* [] memory mapper adds the value of pointer register to the instruction address;
* distribution bus broadcasts the instruction to all gates;
* each gate inspects the instruction;
* matching instruction is passed on to the right worker;
* worker does his job;
* the result enters the collector bus;
* collector bus transports the result back to the mixer.

Given the philosophy of chaining operations, making the mixer wait for the result to come back seems a shame. The mixer should emit a new instruction long before the previous instruction has passed all those steps.

It means that the result of previous operation isn't yet available by the time the next opcode is processed by the mixer.

It's not really a problem, because if the next opcode requires an argument, then the mixer would block and wait for that result to arrive, so the program will work as expected anyway. But it would waste precious ticks: a lot of littlemen would be stalled when they could've been working on the next instruction.

A good program should account for such laggy results and put opcodes in such order to minimize the stall.

For example, to put a pixel of color `10` at position `(5, 2)` on the display, one might write this hopefully understandable code:

```
imm    5
fbx
imm    2
fby
imm   10
fbpix
```

Executing this program would involve a good deal of waiting because each of framebuffer instructions `fbx`, `fby` and `fbpix` would wait for the value produced by previous `imm` instruction.

On the other hand this program:

```
imm    5
imm    2
imm   10
fbx
fby
fbpix
```

makes full use of pipeline processing. This order of statements gives each result of `imm` instruction plenty of time to get back to the mixer before it's consumed by respective framebuffer instruction.

Real LARK CPU measures 154 ticks for the first program snippet vs 116 ticks for the second snippet, which constitutes approx 25% performance gain.

## ALU operations

Certain basic operations can be performed much faster, and their result can be available to the next instruction with no lag noticeable to the user program. Such non-delayed operations are much more convenient for a programmer, so implementing them is well worth the trouble.

For example, loading/storing a value in a CPU register, performing basic arithmetic and logic operations, comparing values, etc. In desktop processors, most of these things are performed by [ALU](https://en.wikipedia.org/wiki/Arithmetic_logic_unit), so for the sake of similarity I'll call this circuit ALU too.

This ALU circuit also takes instructions from the mixer and delivers results back to the mixer:

![mixer_alu.svg](images/mixer_alu.svg?fileId=84015#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

but it does so in more efficient manner.

* It doesn't use as many intermediate buses, thus reducing the lag.
* It uses faster gates that only check for exactly matching opcode, unlike generic device gates that can match opcodes by mask.
* The workers are highly optimized to produce the result as fast as possible, with no irrelevant littlemen operations (like changing the direction) between receiving the instruction and sending the result.

A nice touch here is that results of ALU operations have higher priority than results of common operations performed by general bus devices. This is because the data pipe from the ALU has higher priority for littlemen `R` operation.

For the user it means that when performing a sequence of ALU opcodes, the program should not worry about laggy results from common bus devices. No low-priority result arriving from a bus device at random moment will ever ruin the expected order of ALU results.

It also means that ALU operations seemingly replace the value at the head of the data queue, no matter if the queue contains more data units.

Looking back at the pixel example, if one wants to modify the `x` coordinate of the pixel before sending it to the display:

```
imm    5
imm    2
imm   10
inc
fbx
fby
fbpix
```

that new `inc` instruction will fetch the next value from the data queue (that'll be `5`), increment it by one and put it to ALU (high-priority) data queue. So the following `fbx` instruction will pick the high-priority value `6`, and the other instructions `fby` and `fbpix` will pick their respective arguments from the common data queue, as before.

## Turbo cycles

Not only ALU operations deliver the result just in time for the next instruction, the loops in their littlemen circuits are more tight. Littlemen in ALU return to their ready position faster than littlemen in bus processing devices.

The mixer may emit ALU instructions at higher rate than common bus instructions. The reference implementation of common gate (discussed below in [Implementation](#Implementation) section) takes 18 ticks to process an instruction, while ALU circuits can process an instruction in 12 ticks.

The entire program could work faster if the mixer was capable of producing variable-length cycles depending on whether the current opcode targets the ALU or common bus.

Ok, here's the design that does just that:

![mixer_fast.svg](images/mixer_fast.svg?fileId=84017#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

It contains decoder that inspects the opcode just before it enters the mixer. Opcodes lower than certain factory-configured value are deemed ALU, all the other opcodes are assumed to be targeted at devices on common bus.

The decoder informs the mixer if it should emit a turbo cycle for the next instruction, and at the same time informs the filter if it should block the next instruction from reaching the common bus. Such filtering is required to save slow common device gates from being overloaded by high rate of turbo cycles.

## Program structure

At last, it's time to get back to the source of opcodes that drive the entire processing unit described above.

Like in common desktop CPUs, at the lowest level a program is written in assembly language, where each statement gives one opcode of the compiled program.

Assembly program is abstracted as a collection of code blocks. Code block is a sequence of statements that are always executed, well, as one block. There are no jumps, calls or conditional branching in the middle. Any jump, call or return ends the block.

A typical code block in assembly listing begins with a label that marks the entry point to that block, and ends in either jump, conditional jump, function call or function return statement.

A code block always have only one *entry* at its first statement. Once the block is completed, the control is transferred to one of the other code blocks in the program (or maybe to the same block again, if it's a loop), depending on some condition. The set of possible next code blocks constitutes the *exits* of the given code block.

Most code blocks have only one or two exits, corresponding to convenient conditional branching that can be either taken or not.

LARK CPU generously allows code blocks to have as many as 4 exits, at most. As long as extra exits aren't really used by the program, there's neither performance nor ROM penalty for having so many potential exits.

![code_block.svg](images/code_block.svg?fileId=84011#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

Each code block is assigned an address by which it can be reffered in various branching instructions. The address is assigned at compilation time. Individual instructions within the code block are not addressable.

## Branching concept

What with data processing being the primary goal of CPUs, branching instructions look kind of second-class. They do not process user data, but rather control the flow of "useful" instructions.

In this CPU let's try to keep data processing and branching as separate as possible, and move all branching instructions from the main core to a smaller independent helper core, dedicated only to controlling the flow of the program.

![branching.svg](images/branching.svg?fileId=84012#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

Data processing core is unaware of any code address. It provides the branching core with "decision" -- the index of the exit that should be taken by the current code block, and then the exact address of the next block is determined entirely by the branching core. This saves the data processing core some trouble, as dealing with small fixed integers `0..3` is easier than dealing with arbitrary large code address values.

## Branching instructions

That branching core is fed its own opcodes from its own part of the program ROM. So, each code block stored in ROM actually provides two streams of opcodes: one for the data processing core and another one for the branching core.

![rom.svg](images/rom.svg?fileId=84021#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

There are only 3 branching instructions:

* `goto (Addr) if [new] decision (Mask)`
* `push (Addr) if [new] decision (Mask)`
* `ret if [new] decision (Mask)`

Each branching instructional contains optional `new` flag that indicates whether this instruction should fetch new decision from the processing core, or use the previous decision.

Each branching instruction is conditional: it contains the bit mask of decisions for which it should be executed. Any instruction that do not match the current decision is simply ignored (skipped) with no effect.

An unconditional jump is implemented as

```
goto (Addr) if decision (0, 1, 2 or 3)
```

A typical "jump if not equal" instruction of x86 architecture is implemented as a sequence of two instructions

```
goto (BranchAddr) if new decision (1, 2 or 3)
goto (NoBranchAddr) if decision (0)
```

A typical "call" instruction of x86 architecture is implemented as

```
push (ReturnAddr) if decision (0, 1, 2 or 3)
goto (SubroutineAddr) if decision (0, 1, 2 or 3)
```

A "ret" is a "ret", nothing fancy about it. It just pops the address from the call stack and passes that address on to the ROM.

Hopefully, branching core would work faster than data processing core, so that it advances to the next code block before the data core finishes processing all opcodes from the previous code block, thus providing the data core with a steady stream of opcodes, preventing its stalls.

The reference implementation of the CPU can sustain processing rate as high as 10 ticks per branching instruction, compared to 18 ticks per common bus data processing instruction and 12 ticks per fast ALU instruction.

## Branch prediction

Though the branching core is separate from the data processing core, it still needs decisions generated by the data processing core. Data processing core contains a group of comparator ALU (fast) devices that perform typical comparision operations (equal, not equal, greater than, etc.) and send the result in the form of decision to the branching core. Here's the final schematic of the data processing core:

![mixer_comparators.svg](images/mixer_comparators.svg?fileId=84016#mimetype=image%2Fsvg%2Bxml&hasPreview=false)

Anyway, decision is the point of synchronization that may stall the branching core.

User program may reduce the stall time by providing branching decisions as early as possible, perhaps well in advance of the moment where actual branching would happen.

For example, many loops in contest problems are repeated predictable number of times: the length of input data was usually provided as the first input value.

So, data processing core may well provide an opcode for "repeat decision X N times" (where `N` is taken from the argument of the instruction), thus eliminating the need to reduce loop counter by 1 and compare it against zero on every iteration. One repeated decision will satisfy N branching points for all loop iterations at once.

# Implementation

!!! TBC... Writing it all took much longer than I anticipated... And I've yet to show how those ideas map to the actual implementation of the CPU...

## Particular devices

Some devices that can be attached to the common instruction bus are interesting enough to be described in detail.

### IO controller

### Display controller

### Data stack

### History device