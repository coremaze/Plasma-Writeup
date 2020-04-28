# Appedix A - Opcodes

Opcode 0x05 - A bit confusing, but since the argument is always zero in practice, it pushes 32.

![0x05](Images/Opcode0x05.png?raw=true)

Opcode 0x0C - Stop emulation. Since CALL literally calls the emulator loop again, this can return from a CALL as well. This also removes from an `std::list` of what I later learned were sort of like stack frames. They contain a reference to some offset from the stack so that you can load and store variables. This instruction also, in practice, always takes an argument of negative the number of arguments a subroutine has, but it doesn't affect execution.

![0x0C](Images/Opcode0x0C.png?raw=true)

Opcode 0x12 - Call to subroutine. Literally causes the host to call the emulator again. Includes a bit of pointless arithmetic to confuse a reverse engineer.

![0x12](Images/Opcode0x12.png?raw=true)

Opcode 0x13 - Push string literal. The parsing is too long for an image, but it's still relatively straightforward. It reads `<argument>` dwords, converts the dword array to a char array, puts it in a vector, puts the vector on the stack, and returns the number of dwords read.

Opcode 0x14 - Compare whether the first element on the stack is greater or equal to the second element.

![0x14](Images/Opcode0x14.png?raw=true)

Opcode 0x17 - Add two elements on the stack. Implementation already described.

Opcode 0x1B - Load variable from the stack, using an offset from the stack frame if positive, or from the base of the stack if negative.

![0x1B](Images/Opcode0x1B.png?raw=true)

Opcode 0x24 - The implementation was very confusing for me, but based on the context, using the disassembler I was building simultaneously, I guessed that it stored a variable from the top of the stack to some offset from the stack frame, and I seemed to have been correct.

![0x24 Disassembly](Images/Opcode0x24-1.png?raw=true)

![0x24](Images/Opcode0x24-2.png?raw=true)

Opcode 0x2C - Create a new stack frame. This implementation was also very confusing for me, so I guessed, but this also seems to be correct.

![0x2C](Images/Opcode0x2C.png?raw=true)

Opcode 0x3A - Removes a number of elements from the stack.

![0x3A](Images/Opcode0x3A.png?raw=true)

Opcode 0x3B - Boolean AND. Implementation already described.

0x43 - Push literal to stack. Also has useless arithmetic going on.

![0x43](Images/Opcode0x43.png?raw=true)

0x44 - Push literal to stack, just like 0x43. In practice, it seems to be used to push return values to the stack at the end of subroutines, but it does the same thing.

0x4A - Multiply.

![0x4A](Images/Opcode0x4A.png?raw=true)

0x56 - Divide.

![0x56](Images/Opcode0x56.png?raw=true)

0x5C - Modulo.

![0x5C](Images/Opcode0x5C.png?raw=true)

0x63 - Subtract.

![0x63](Images/Opcode0x63.png?raw=true)

0x69 - Compare if equal, with more multiplication involved to confuse.

![0x69](Images/Opcode0x69.png?raw=true)

0x71 - Compare if lesser.

![0x71](Images/Opcode0x71.png?raw=true)

0x73 - Branch of zero, and get floats involved to confuse. Argument is relative to PC after instruction executes.

![0x73](Images/Opcode0x73.png?raw=true)

0x74 - Load pointer of element.

![0x74](Images/Opcode0x74.png?raw=true)

0x75 - Compare if zero.

![0x75](Images/Opcode0x74.png?raw=true)

0x86 - Branch. Argument is relative to PC after instruction executes.

![0x86](Images/Opcode0x86.png?raw=true)

0x89 - System call. This calls into a predefined ser of x86 code somehow. Its implementation is complicated, and in practice, I just followed its execution to figure out what syscall numbers went to where.

This left me with a table of all the opcodes I'd need to know:

![Opcode Table](Images/OpcodeTable.png?raw=true)