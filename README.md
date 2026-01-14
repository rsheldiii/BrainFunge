# BrainFunge
 A [BF](https://en.wikipedia.org/wiki/Brainfuck) interpreter written in Befunge!

 * Funge-93 compliant!
 * Turing complete (with minimal changes) in Funge-98!
 * Tested and working [here](https://www.jdoodle.com/execute-befunge-online/), [here](https://www.tutorialspoint.com/compile_befunge_online.php), and [here](https://esolangpark.vercel.app/ide/befunge93)!

```
v
0++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+
0++.------.--------.>>+.>++.
v
v
v
v
v
>::44*5*1-%1+\44*5*1-/1+g02g68*-!v                                              
^                  v     !-*86g10_99*9+1+-:v                                    
^                 v_99*9+3+-:v                                                  
^                             <-1p20+g20!-2_$02g1-:02p68*-!!2*-v                
^              <+1 p10+g10!+2_$01g1-01p>                       v                
^                 >           76*1+-:v                         v                
^                                v:-1_$\:1+:0g1+\0p\>    #+    v                
^                            v:-1_$\:1+~84*+\0p\>        #,    v                
^                        v:-1_$\:1+:0g1-\0p\>            #-    v                
^                  v:-*27_$\:1+0g84*-,\>                 #.    v                
^              v:-2_$\1-\>                               #<    v                
^      v:-+1*56_$\1+\>                                   #>    v                
^  v:+2_$\:1+0g84*-!v                                          v                
^                 v\_\02g1+02p2->                        #]    v                
^ @_$\:1+0g84*-v  #                                            v                
^            v\_\01g1+01p>                               #[    v                
^+1<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<                
```

## BF?

BF is an esoteric programming language consisting of 8 instructions (`>` `<` `+` `-` `.` `,` `[` and `]`) that is a fantastic introduction to writing interpreters / compilers or proving a system is Turing complete. The [Wikipedia](https://en.wikipedia.org/wiki/Brainfuck#Language_design) article is a great place to read more.

## Befunge?

Befunge is an esoteric programming language designed to be incredibly difficult to compile - it features (in the funge-93 specification) an instruction pointer that can move in 2 cardinal directions, an 80x25 grid of instructions, and the ability to modify its own code - aka "reflection". Once again the [Wikipedia](https://en.wikipedia.org/wiki/Befunge) article is a great place to learn more!

### Is this actually turing complete then?

Almost. With the limited code and data space, running in a funge-93 compliant compiler / interpreter, this code is not turing complete. It could be if the program was fed in via stdin, but I didn't want to interleave program and user input.

It is, however, trivial to achieve turing completeness in a funge-98 compiler / interpreter while still using the Funge-93 instruction set. The text wrapping code should be removed, after which the BF program is written entirely on line 1. Funge-98 programs are not constrained to any size grid, so data and program space can stretch continuously, and turing completeness is achieved.

### This doesn't look very space and/or time efficient

It probably isn't! :D

# Architecture

There is not much to a BF interpreter. You need:

1. An array to hold onto your data
2. An instruction pointer, to point to the current instruction
3. A data pointer, to point to the current data

We also avail ourselves of two counters: one each for the amount of left and right brackets we are abiding by at a given moment. No other state is needed; all other commands are executed immediately, using the topmost row of the grid as the data storage.

## Brackets?

BF accomplishes looping by using brackets: if a right bracket is encountered, and the data at the current data pointer is nonzero, the program loops back to the matching left bracket. Vice versa is also true, if a left bracket is found and the data at the data pointer _is_ zero, we jump forward to the matching right bracket. At first blush this can look tricky to implement but luckily, bracket matching is O(n). When moving forward you keep a tally of how many left vs right brackets you've seen; when that number reaches 0, you have reached the end of the original bracketed data. When moving backward, the tally becomes right vs left brackets.

This bf interpreter uses two "static" counters to achieve this, that physically occupy a space on the board. These could be held in memory, but Befunge only supports swapping the topmost two values in the stack. By storing the bracket counters on the board itself we need only store the data and instruction pointer in memory, making swaps trivial. What's more, the bracket tallys reuse the original instruction and data pointer initialization instructions as a space-saving measure!

# Code Overview

You don't need to know the entire instruction set of Befunge to make sense of this section, but do note that:

1. All Befunge programs start in the top left of the board and travel right unless directed otherwise
2. The instruction pointer's direction is changed via `<`, `>`, `v` and `^` - hitting a cell with one of these arrows in it will unconditionally change the direction of the instruction pointer
3. Befunge is a stack-based language - most functions either pop a value onto the stack, manipulate values on the stack, or do both

## Initialization

 We immediately redirect the instruction pointer downwards, and pop two 0's onto the stack; these are the data and instruction pointer x values. Both values are actually offset by 1, but we just accomplish this by adding 1 to a copy of the pointer value whenever we need to reference something. These two zeros are the instructions that get repurposed into the bracket tallies.

## `>::44*5*1-/1+\44*5*1-%1+`

Yeah... Befunge is a bit of a write-only language.

This code used to be much simpler: `:1+1g` -> copy the instruction pointer, add 1 to it, add a y value of `1` to the stack, and `get` the value at the square [x+1,1]. However, the funge-93 specification clearly outlines a maximum board size of 80x25 instructions, and most Befunge interpreters / compilers dutifully comply. "Hello World" in bf is well over 80 characters, so I needed to write some text wrapping code in order to have any usable amount of program space. There is a Funge-98 specification which allows for arbitrarily sized program spaces, but it also allows for 3-dimensional program spaces, so it's a bit overkill.

This code copies the instruction pointer twice, then modulos and then divides the instruction pointer by 79 to find the x and y values of the instruction pointer from the instruction pointer value. We add 1 to both x and y because of the offset we've mentioned earlier. If you've ever represented a 2d array in a single array, that is exactly what we're doing here.

Befunge uses postfix arithmetic, so `44*5*1-` means `4 * 4 * 5 - 1`. `\` is the `swap` command, and swaps two values on the stack; in this case we're swapping to a fresh copy of the instruction pointer value to find the `x` coordinate of the current instruction after finding the `y`.

## `g02g68*-!v`

Our stack currently consists of the data pointer, the instruction pointer, and the x and y values of the next instruction, in that order. we use `g` to grab the ascii value of the next instruction. We then immediately grab the value of the right-bracket tally - if we are in "bracket searching" mode we only process brackets, so we need to short circuit here.

`g` returns the ascii value of the character at the given location, so `0` actually returns `48`. We could use empty spaces instead of overwriting the two pointer initialization values, but most Befunge interpreters interpret blank instructions as `space`, or `32`. You might be able to find an interpreter that lets you input a `NULL` character, but I opted to just subtract 48 from each tally instead.

`!` is the `not` instruction and is used here just to flip the direction of the branching conditional, explained below

## `v     !-*86g10_99*9+1+-:v`

This line is best read in two parts: everything before and everything after the underscore. Underscore (and pipe) in Befunge are branching conditionals; if the current value on the stack is 0 the instruction pointer moves right, if not it moves left. If the instruction pointer moves right it means we've seen a right bracket. We check if the instruction we just accessed is a left bracket (91 -> `9 * 9 + 9 + 1`) by subtracting its value, and feeding into another conditional. We duplicate the subtracted value just before we compare, for reasons we will discuss below.

If we move left, it means we haven't seen any right brackets. We immediately check if we've seen any _left_ brackets then, using the same logic as before. Because we're now moving right-to-left, the instructions to do so are reversed.

## `v_99*9+3+-:v`

Continuing where we left off, if we move left from the underscore it means we haven't seen any left brackets either; we immediately exit the bracket-matching code.

If we move right it means we've seen a left bracket, so we check if the current instruction is a right bracket (93).

## `<+1 p10+g10!+2_$01g1-01p>`

Finishing where we left off, if we move right it means we've seen a left bracket and the current instruction is a right bracket. We discard the duplicate subtracted value, and subtract one from the left bracket counter, then exit the instruction loop. We have effectively exited this bf loop!

If we move left, it means we've seen a left bracket but this current instruction is not a right bracket. We need to keep track of _all_ the left brackets vs right brackets we see however, so we quickly add 2 to the subtracted value to see if it was actually, in fact, a left bracket (91) instead of a right bracket (93). If the value is `0`, `!` will turn it into a `1`, so we not the value and then add it to whatever we have in the left bracket tally spot.

We are now left with a stack with only the data and instruction pointer remaining. We add 1 to the instruction pointer to go to the next instruction, solely because the code to do this is squirreled away in the bottom left corner and I couldn't figure out a way to access it that saved either space or instructions.

## `<-1p20+g20!-2_$02g1-:02p68*-!!2*-v`

Cast your mind all the way back to 3 sections ago, we have seen a right bracket and just subtracted the value of a left bracket from the current instruction to see if they are the same. If they _aren't_ the code is very similar: we check if the current instruction might be a right bracket instead, add it to the tally if so, but otherwise _decrease_ the instruction pointer. If we have seen a right bracket, but are not currently visiting a left bracket, we always travel left in our bf program.

If the current instruction _is_ a left bracket however, things become a little more tricky. We discard the duplicate subtracted value as before and subtract one from the right bracket tally, but then we duplicate that value just before we put it back, and subtract 48 from it. we then `not` it twice, multiply by 2, and subtract that value from the instruction pointer.

What we're doing here is using the current value of the right bracket tally to determine what direction to send the instruction pointer. `!!1` = 1, but so does `!!2`, `!!3` etc. `!!0` however is 0, and 2 * 0 = 0. If we have more than 0 right brackets left, we subtract 2 from the instruction pointer; this counteracts adding one later on in the bottom left corner. If we have 0 right brackets left we have exited traveling backwards across our code to the matching left bracket, and all this code is a no-op, with the effect of moving the instruction pointer forward one instruction.

Congratulations, the hard stuff is over! now to get to the actual commands themselves.

## `76*1+-:v`

Short and sweet, this code subtracts 43 from the instruction value and duplicates it. `+`'s ascii value is 43, so we're checking if the current instruction is a `+` on the next line.

## `v:-1_$\:1+:0g1+\0p\>`

This and 6 of the lines after it will perform a specific bf instruction if the top of the stack is 0 (ie the character matched); otherwise, they will subtract a number from the instruction value and move down the chain. This first instruction is tasked with increasing the value of the data at the data pointer by 1. Each right conditional branch discards the instruction value as we no longer need it; we then swap to the data pointer, duplicate it, add the offset and duplicate it _again_. We add 0 as the y value of the data pointer and call `g` to get the value currently stored. We add 1 to it, swap the value with the duplicated value of the data pointer and pop 0 onto the stack, so our stack looks like [instruction pointer, data pointer, data value, x, y]. we call `p` to `put` the data value at coordinates `[x,y]`, then swap one more time to put the data pointer back at the bottom of the stack. Phew!

`#` is the `jump` command and just jumps over the next code point. the aligned blocks are just sugar to show what bf instruction was just executed.

If we instead went left it means we didn't match on the character. we subtract one to get to `44` (,) and move down the chain

## `v:-1_$\:1+~84*+\0p\>`

Here we execute the `,` bf command, which inserts user data into the current data location. In Befunge we copy the data pointer, add the offset, ask the user for input (~) and add 32 to that input, then place it in the correct data position. Since most Befunge interpreters interpret `get`ting an empty instruction as returning a space, we just 32-index all of our bf data. neat huh?

## `v:-1_$\:1+:0g1-\0p\>`

The `-` command subtracts from the current data in the current data location. The code is identical to two sections above, save that we subtract one from the data value instead.

## `v:-*27_$\:1+0g84*-,\>`

The `.` command outputs whatever data is at the current data location as an ascii character. We do this by grabbing said data, subtracting 32 from it, and outputting it with `,`

## `v:-2_$\1-\>`

Getting the hang of it? What's this one do?

`<` in bf moves the data pointer left. We accomplish this by subtracting one from the data pointer, which as you remember is on the bottom of the stack, so we must swap to it (and from it).

## `v:-+1*56_$\1+\>`

`>` in bf moves the data pointer to the right. The right branching conditional is almost identical to the section above. The left branching conditional looks different, but we're just subtracting 31 instead of 1 from the instruction value; we're checking if the instruction was a right bracket on the next line

## `v:+2_$\:1+0g84*-!v`

Here's where we can enter a right-bracket loop. We grab the data at the data pointer and subtract our 32 offset; if the data is _nonzero_, we'll proceed to another code block. If instead we were passed a nonzero value to the underscore, we _add_ two to the value, as left bracket has an ascii code of 91 compared to the right bracket's 93. I could have checked the left bracket first, I was just too lazy to refactor it after some funky logic I was doing got factored out.

You might have noticed we `swap`ped, but never swapped back. We'll rectify this below

## `v\_\02g1+02p2->`

Here's that extra code block. Regardless of our branch we swap the data and instruction pointers back to their rightful places. If we do not match (data at data pointer was 0) we harmlessly proceed to the next instruction, ignoring the right bracket loop. If we _do_ match, we add 1 to the right bracket count and subtract 2 from the instruction pointer in the same way we did previously, since the bottom left corner will add 1 later.

## `@_$\:1+0g84*-v`

We actually deviate from the bf specification in one significant way: instead of ignoring comments, we terminate on any non-bf instruction. The reasons for this is twofold:

1. We don't really have enough space for comments, but more to the point
2. We can't tell when the program ends

There is no end-of-input character; we could add one, but that is effectively adding a new instruction to BF, which feels a lot worse than just terminating on non-bf input.

Anyways, if we don't match on the left bracket's ascii value we immediately terminate the program. If we do match, we grab the data at the data pointer and subtract our 32 offset...

## `v\_\01g1+01p>`

...Which, if zero, has us add 1 to our left bracket tally just like we did two sections ago for the right bracket tally.

# That's all folks!

Thanks for reading!
