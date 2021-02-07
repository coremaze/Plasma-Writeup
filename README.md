# Restoring Picroma Plasma Without Patching it

## and Reverse Engineering an Unknown Instruction Set

### Introduction

Picroma Plasma is a vector image editor written by Wolfram von Funck under the company Picroma. Some videos of its usage are available [on Picroma's YouTube channel](https://www.youtube.com/user/Picroma/videos).

Picroma/Wollay is also the author the game [Cube World](https://www.cubeworld.com/), and as a result, the game's GUI is constructed using Plasma graphics. I became interested in these Plasma graphics files in 2016, as I was beginning to get into reverse engineering and programming, and I had just finished building a converter for another, much simpler, undocumented image file format. Until this point, no one else had managed to make any useful modifications to Cube World's image files.

The years went on, and with the help of my friend [Andoryuuta](https://github.com/Andoryuuta), I created [PLXML](https://github.com/ChrisMiuchiz/Plasma-Graphics-File-Parser), a tool that could convert the Plasma graphics files (known as .PLX, .PLD, or .PLG) to and from an XML file format. However, since these are vector graphics, it's more difficult to create an editor for them, and there was no GUI based editor except Plasma itself.

Plasma was Picroma's first (and probably, in their eyes, their primary) product, but only one release was ever created, and it was in 2011. It used an authentication server which eventually went down, so when it stopped working, most people just got rid of the software and moved on. It wasn't until April 20th, 2020 that the installer from 2011 resurfaced and we could get to work on making this old art tool work again. 

Note: The installer is PicromaPlasmaSetup.exe (SHA256 `1ED1DA68781666CC03DAD79325B1F0603552DEDF55357778DA8518DFBF54F37A`).

### Digging in

So, where do we start? When Plasma is launched, it wants to contact the authentication server at picroma.de using HTTP, and if you try to login or activate the product, it does not work:

![Could not connect](Images/NeedAuth.png?raw=true)

Okay. The most obvious next step is to find the code responsible for "activating" Plasma and just patch some code to make it always activate, no matter what the authentication server does. 

Investigating the code in the program is done using a disassembler, with my preferred one being the historic industry standard, [IDA Pro](https://www.hex-rays.com/). Free alternative disassemblers include [Ghidra](https://ghidra-sre.org/) and [Cutter](https://cutter.re/). These are also capable of producing "decompiled" C-like code. The eager should beware, as "decompiled" native code is not always functionally accurate and always requires significant human assistance in order to produce good output. That being said, moving forward I will show a combination of disassembly and decompilation, as the decompiled code is easier for a reader to digest and is more succinct.

![Assembly](Images/Assembly1.png?raw=true)

There are a couple places in the binary that reference a few data members in order to determine whether the program has been activated. However, any attempts to patch these jumps or write affirmative values to these data members will result in the program crashing. Usually, this is in the construction of a `Sheet` object (Note that [RTTI](https://en.wikipedia.org/wiki/Run-time_type_information) was in this binary, so some class names can be recovered), the parent class of `DesignSheet`, but patching around that just results in more unhandled exceptions. It is not possible to patch the program in this manner in order to get it to work. Let's take a look at the code which handles bad (or a lack of) responses from the server:

![Error handling](Images/ReportError.png?raw=true)

This entire function is dedicated to error handling. I set up my own server with Flask to play with how it responds to various inputs, but I wasn't able to get anything interesting to happen other than change which error message it displayed. It can attempt to access any of the following paths:

```
/LS/Activation/Logout/
/LS/Activation/Activate/
/LS/Activation/Login/
/LS/Activation/Validate/
/LS/Activation/Deactivate/
```

It also seems to be able to contact `/Download/Plasma/release.xml` for updates, but as of the time of investigation, this was not yet valuable to me. I *did* end up using it, supplying an update with the primary purpose of fixing bugs to increase system compatibility so this software can be more effectively preserved.

While watching outbound traffic, I noticed that Plasma was also sending some parameter called `id` formatted as a hexadecimal string like so:

`/LS/Activation/Activate/?id=393A3145362D8435C07146B46343695B466D3149735D9467BE5550C345877692418D315744E6A276D4774DE439BA9AC04698316930718588D195734331CD84F230CDF174D094849700CB42BB38FA842434DF1E8AFCB490B33EE653C3301AB055440C2`

At this point, I had no idea what to do with it, but I assumed it represented the machine and serial in some way. 

Let's look for xrefs (cross-references) to this error handling function to see what triggers it.

![Error handling](Images/ErrorCatch.png?raw=true)

Ah, so that error handling function is called if there is a `plasma::IOException` raised while handling the response. This is a type of C++ exception created by the developer which is being gracefully handled.

I started placing breakpoints around to see which function raised the exception, and I found it happening in quite a peculiar function...

![PLX Parser](Images/PLXParser.png?raw=true)

I recognize this code from reverse engineering Cube World to create [PLXML](https://github.com/ChrisMiuchiz/Plasma-Graphics-File-Parser). This is code to parse a Plasma graphics file. It seems that, for some reason, Plasma is expecting a Plasma graphics file to be sent back from the authentication server. Since it is failing to parse my response, it is triggering the error handling code.

The function (which calls both the parser and error handler) receives data contained within an `std::vector*` argument:

![Data vector](Images/DataVector.png?raw=true)

However, examining the data contained within the vector reveals nonsense, even when my server gives it a valid PLX file. The vectors coming into this function are the correct size to be the data being sent by my server, but the data is not what I sent. Upon more experimentation with small buffers, it seemed to be (de)obfuscated by performing addition/subtraction on the bytes and swapping them around somehow, so it didn't seem like a particularly strong obfuscation algorithm.

For now, I decided to just overwrite the vector in memory with untouched data at this point in execution. This effectively bypasses whatever obfuscation algorithm is happening. After implanting this valid PLX data, I stopped getting error messages when attempting to authenticate, but it caused exceptions instead. This ended up being due to elements being missing from my constructed PLX file and causing null pointer dereferences. A lot of these issues are caused by looking for an object by name which does not exist in the PLX:

![Widget by name](Images/WidgetByName.png?raw=true)

I worked through these exceptions until I constructed a PLX file using [PLXML](https://github.com/ChrisMiuchiz/Plasma-Graphics-File-Parser) with enough of the bare minimum requirements to load up one of Cube World's PLX files.

![Cube World logo](Images/CubeWorldLogo.png?raw=true)

It's not pretty, but this is probably the first time anyone's been able to use Plasma at all in the better part of a decade. It seems that Wollay removed a critical UI file (for the sheet which artwork exists on) from Plasma, and made it so that the server would provide an obfuscated version of it to the client. That way, no amount of tampering could get an unauthorized copy of Plasma to work. Unfortunately, without the authentication server, authorized copies of Plasma cannot work anyway.

Around this time, I started looking at what the picroma.de domain used to point to. I didn't find much of interest on archive.org, but...

**The domain was now available after all these years, and I bought it.**

### No Need For a Patch

With ownership of the picroma.de domain, in theory I could make Plasma work without even touching the binary or modifying the hosts file. Just, vanilla copies of Plasma would suddenly start working again one day. The thought of this was so cool to me that I halted my pursuit of creating a better sheet PLX file and started a deeper investigation into how the response was getting (de)obfuscated by the client. Understanding this would hopefully allow me to have my server send well-formed obfuscated responses that an unmodified client could understand.

I had previously not been able to understand anything about how the deobfuscation algorithm was actually implemented. I found the last bit of code that seemed to touch it before it was transformed:

![Unmodified buffer](Images/UnmodifiedBuffer.png?raw=true)

However, setting breakpoints on bytes inside this buffer to see what algorithm was touching it didn't lead me to anywhere useful. I tried to work my way up the stack to find the loop that was altering the buffer, but I was always led to a function that didn't seem to be responsible for deobfuscating a buffer, and it also didn't make very much sense to me since it was constantly calling functions by reference.

![Weird Loop](Images/WeirdLoop.png?raw=true)

What was happening here seemed over my head. I'd seen this same type of code pattern in Cube World before, and was unable to figure out what was going on.

Apparently Andoryuuta was less oblivious, because he noted he wasn't able to figure out how the virtual machine in Cube World worked.

Then I realized that's what I was looking at.

### An Embedded Emulator

When framing it in the perspective of some kind of bytecode interpreter, emulator, or virtual machine, and given that I have created an emulator for obscure hardware before, this code suddenly made a lot more sense. So, with that bit of knowledge, I started working on the decompilation of this strange loop function:

![Emulator Loop](Images/EmulatorLoop.png?raw=true)

This machine, whether to be called a virtual machine, emulator, or bytecode interpreter, had some `std::vector` containing code that was slightly obfuscated along with another `std::vector` which, when indexed using an opcode, contains a function pointer corresponding to that opcode. Data is simply deobfuscated by performing `offset - program[offset]` . Eventually, the opcode would become `0x0C`, and the emulation would stop, so this is likely a HALT or RET opcode. Each piece of data in this machine is 4 bytes long. Each instruction contains 1 opcode and 1 argument. When the function corresponding to an opcode is called, it can return a value that is added to the emulator's program counter. This is used for branching and in case an instruction needs to read more than just its argument.

I dumped this strange bytecode from memory, and knowing that `data = offset - program[offset]`, I was able to deobfuscate the entire thing, but that isn't much use without some kind of disassembler.

With that, I decided I had to figure out what each opcode did. It is worth noting at this point that I tried to see if this bytecode belongs to any existing CPU or language, but my search came up short. The constructor of this emulator conveniently assigned all the functions for me to analyze, but the list went on and on...

![Lots of opcodes](Images/LotsOfOpcodes.png?raw=true)

I painstakingly assigned meaning to every opcode function that I could, and I identified that this was a completely stack-based machine which did all its operations using the stack. It's worth noting that every element on the stack is itself a vector. This allows any element to hold arbitrary data, such as pointers, strings, or simple integers. Here's an example of addition:

![Addition](Images/Addition.png?raw=true)

It takes two elements off the stack, adds them together, and then pushes the result back to the stack.

Unfortunately, you can see a hint of something that became far too common while analyzing these opcodes. It makes one element negative before subtracting it. This is part of a larger pattern of fairly weak attempts to confuse a reverse engineer that made it frustrating to figure out what all the opcodes did, and there were many duplicate opcodes that were just implemented in different ways.

Here's addition implemented by multiplying the result with some number and its reciprocal:

![Confusing Addition](Images/ConfusingAddition.png?raw=true)

Here's doing addition but with extra recursion:

![Recursive Addition](Images/RecursiveAddition.png?raw=true)

Multiplication by multiplying by 2 and then later dividing by 2:

![Multiplication with Division](Images/MultiplicationWithDivision.png?raw=true)

Boolean ANDing by multiplication:

![Boolean AND by multiplication](Images/BooleanAndByMultiplication.png?raw=true)

Arithmetic that doesn't do anything:

![Useless Arithmetic](Images/UselessArithmetic.png?raw=true)

Warming up the PUSH machine by pushing and popping to and from the stack a few times first:

![Push Pop](Images/PushPop.png?raw=true)

The list goes on.

There was one instruction that stopped me from blindly trying to disassemble every pair of dwords. It was the only one which was variable in length, so it caused the `<opcode> <argument>` alignment to change. It's an instruction to push a string literal to the stack, and it's formatted `<opcode = 0x13> <argument = number of characters in string> <string, with each character taking 4 bytes>`. 

With this knowledge, I decided to take a different approach. I suspected that only a few of the implemented opcodes were actually used, so I parsed through the bytecode, accounting for alignment changes due to the PUSHSTR opcode, and generated a set of 24 opcodes that were actually used:

`[0x05, 0x0C, 0x12, 0x13, 0x14, 0x17, 0x1B, 0x24, 0x2C, 0x3A, 0x3B, 0x43, 0x44, 0x4A, 0x56, 0x5C, 0x63, 0x69, 0x71, 0x73, 0x74, 0x75, 0x86, 0x89]`

This is a much more manageable amount of work.

See [Appendix A - Opcodes](Appendix_A_-_Opcodes.md) for more detail on individual opcodes.

Reverse engineering these left me with a table of all the opcodes I'd need to know:

![Opcode Table](Images/OpcodeTable.png?raw=true)

This is just what I needed to move on to disassembling the bytecode.

### Disassembling a New Instruction Set

I had been working on a disassembler while reverse engineering the opcodes, but now that I have a complete description of the opcodes this program uses, I can disassemble the bytecode and begin to make a serious attempt at understanding its contents. The source for my disassembler is available at https://github.com/ChrisMiuchiz/PLASM-Disassembler.

To find which function was obfuscating the PLX, I set a breakpoint on the data, and once it was written to, I set another breakpoint inside the emulator so that I could identify the program counter at the next virtual instruction.

![Disassembly 1](Images/Disassembly1.png?raw=true)

Note: The comments documenting these syscalls were added later. My strategy for figuring out what each one did mostly came down to simply stepping through to see what functions each one called. Syscalls are not OS syscalls; they are syscalls unique to the implementation of this emulator and are used to interact with x86 code.

I had arrived in the middle of the obfuscation process. However, I wanted to make sure I started from the beginning. This function starts at address `0233`, so I looked for instances of `CALL $0233`. There were a few usages of it, but I had an idea of what part of code I was looking for, so eventually I was led to subroutine `0ED1`.

![Disassembly 2](Images/Disassembly2.png?raw=true)

I started by reverse engineering all the functions that this function called. My strategy for doing this was to look at every chunk of code that resulted in a call, syscall, or variable write, and convert it into a line of higher level pseudocode. For example, this is what the beginning of the `0233` subroutine looked like after I had taken my notes on it:

![Disassembly 3](Images/Disassembly3.png?raw=true)

The decompilation, by hand, of my first function looked like this:

```
sub_0233 (arg0, arg1) {
  var2 = arg0.length;
  var3 = arg1.length;
  arg0.resize(var2 + var3);
  var4 = 0;
  while (var4 < var3) {
    arg0[var2+var4] = var1[var4];
    var4++;
  }
  return 0;
}
```

It appends the data held by arg1 to the data held by arg0. I replaced all calls to this with `CALL EXTEND_ARRAY`. I did similar renaming for future functions as well.

I moved onto the next required function at `056F`:

![Disassembly 4](Images/Disassembly4.png?raw=true)

A nice, short function which easily decompiled to the following:

```
sub_056F (arg0, arg1, arg2) {
  return (arg2[arg0 % arg2.length] + arg0) % arg1;
}
```

This function's purpose was confusing at first, but later, in context, I learned that it was used to generate an index to swap bytes with in order to scramble the PLX. Rather, in this case, it's supposed to be *unscrambling* the PLX, but since it's already unscrambled, it ruins the PLX. It takes an index, the length of the buffer to be used for, and an array of values for use as a key.

I used the same strategies to decompile `0593`:

```
sub_0593(arg0, arg1) {
  var2 = arg0.length;
  var3 = var2 - 1;
  while (var3 >= 0) {
    var4 = sub_056F(var3, var2, arg1); // Get swap index
    arg0[var3] = arg0[var3] - var3;
    var5 = arg0[var3];
    var6 = arg0[var4];
    arg0[var3] = var6;
    arg0[arg4] = var5;
    var3--;
  }
  return 0;
}
```

This, combined with the previous function finally begins to expose the (de)obfuscation algorithm, which is actually a form of weak symmetrical encryption. It takes a buffer and a key, goes through all the indices in reverse (since this is for decryption), performs arithmetic on a byte and swaps it with another byte using said key.

I could move on to the top-level function now, `0ED1`.

```
sub_0ED1(global) { // A buffer representing your "id" is already on the stack
  var0 = new vector();
  var0 = PLX;  // Retrieved through some syscall
  var1 = new vector();
  var2 = new vector();
  var1.resize(16);
  var1 = {37, 124, 3, 213, 190, 48, 142, 34, 7, 77, 143, 210, 201, 97, 86, 23};
  var2.extend(global);
  sub_0593(var2, var1); // decrypt
  sub_0593(var0, var2); // decrypt
  //don't think I need to care about the rest of this function
}
```

A key is constructed at runtime to decrypt your machine's "id" which is already on the stack, and then the id is used to decrypt the PLX.

Finally, we have arrived at the answer for how the decryption algorithm works, and it's trivial to run it in reverse to derive the encryption algorithm that our authentication server will need. However, this takes an encrypted version of our id, and it looks nothing like the hexdump that is being provided as a parameter. Even when decrypted, it looks something like this:

This is all ASCII:

`000000654C1456387226FEC3152EFBC53416A635B21976FFA5D1A844EEA1EE6266D772CEFEF842DE3133C9E20000000801D617654E45521E01D6176537D2672E01D617595D0CF02201D61CCF4C45DFCA`

For my purposes, I don't need to know how the id itself is generated (though, strings in this bytecode reveal that it uses WQL to get serial numbers and such from hardware components), but I do need to know how to go from the parameter being sent to the server to this plaintext form so I can encrypt the PLX with it.

I found where the parameter version of the id was used:

![ID 1](Images/ID1.png?raw=true)

However, that function was called by reference, and returned to one of the emulator's SYSCALL implementations:

![ID 2](Images/ID2.png?raw=true)

That means it's time to go back into the emulator's disassembly.

![Disassembly 5](Images/Disassembly5.png?raw=true)

It constructs a different key than before: `[  4,  21, 132,  64,  32, 132, 243, 132,  17, 177,  43, 132, 101,  42,  44, 150]`

Afterwards, it appends your machine id to another array. It was confusing in its implementation for me, but at runtime, I could see that this `UNKNOWN?` chunk actually fills var1 with your serial. As a result, this concatenates your serial with your machine id. Once this concatenated string is created, it encrypts it using the new key, dumps it as hex, and sends it to x86 code to be used as a parameter for the server.

### Accomplishment

Now we know everything we need to know in order to reverse this process so that the server can generate encrypted data.

Excerpt from server.py:

![Python 1](Images/Python1.png?raw=true)

Excerpt from encryption.py:

![Python 2](Images/Python2.png?raw=true)

This works. Using this code to generate responses from our server allows Plasma to successfully decrypt the traffic. The product activates. We haven't made any changes to the program nor our hosts file. It's a crackless crack.

![Activated](Images/Activated.png?raw=true)

### Finishing up

We have a working authentication server, but the PLX file we're sending still doesn't work quite right. It doesn't look like the sheet displayed in those Picroma videos, and I've had trouble actually adding shapes to the PLX. The worst problem is that sometimes it still crashes Plasma.

I tracked the crashing issue down to attempting to deallocate uninitialized memory. With some experimentation, I realized that in my PLX file, I had 2 `plasma::Nodes` associated with 1 `plasma::Widget`. This was probably causing an attempt to destruct the widget twice. Removing the widget from one of the nodes fixed the issue. Associating a new widget with one of the nodes causes an infinite loop, so I couldn't do that.

While fixing crashing issues, I noticed that if a PLX file that crashes Plasma was sent, Plasma would likely crash next run without even connecting to the server. It turned out that Plasma saves data to `C:\ProgramData\Picroma\Plasma\config` and `C:\ProgramData\Picroma\Plasma\settings`. `config` contains your serial and a value representing the state of your activation. `settings` contains the last encrypted PLX which Plasma received. Additional, less interesting data is stored in the `C:\Users\<user>\AppData\Local\Picroma\Plasma` directory.

I'm not sure how sheets are supposed to work in Plasma, but they seemed to have been rectangles around 1280x720 pixels, with a drop shadow behind them, upon which artwork would exist while being edited. For now, I made them slightly larger rectangles to account for modern monitors. I'm not sure if they perfectly replicate the original functionality, but they seem perfectly usable for now. I will revisit them if I need to.

The most desirable solution would be to get the original sheet PLX. Although unlikely, since Plasma saves the encrypted file to `C:\ProgramData\Picroma\Plasma\settings` upon an activation attempt, it could be possible to recover it from an old installation and attempt to decrypt it. A zero byte file will be saved if authentication fails, but the encrypted data may still exist on the hard disk since the data was not strictly overwritten, so recovery may still be possible. In order to ensure that any remaining copies from 2011 do not accidentally destroy the data by authenticating with my server, I am limiting valid serials to `0000000000000000` and `PlasmaXGraphics`. The latter is used by Picroma to save and obfuscate their PLX files, so it is convenient to be able to use that serial. It is possible to use non-numeric serials by modifying `C:\ProgramData\Picroma\Plasma\config`. This section will be revised if an encrypted copy of the file is recovered.

I'm no artist, so to show it off, here's recoloring Cube World's logo.

![Cube World's normal logo](Images/CubeWorld1.png?raw=true)

![Editing Cube World's logo](Images/CubeWorld2.png?raw=true)

![Cube World's new logo](Images/CubeWorld3.png?raw=true)