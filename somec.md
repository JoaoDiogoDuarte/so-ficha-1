### Notes on Exercise 1

#### Assembly output

This command will output the generated assembly from the C code.
Assembly files has the extension `.s`, which stands for **source** (in the 60s and earlier, the only source code you had were assembly files).

```asm
        .file   "hello.c"
        .text
        .section        .rodata
.LC0:
        .string "Hello World"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        leaq    .LC0(%rip), %rax
        movq    %rax, %rdi
        call    puts@PLT
        movl    $0, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   main, .-main
        .ident  "GCC: (GNU) 15.2.1 20250813"
        .section        .note.GNU-stack,"",@progbits
```


You have a variety of labels that annotate the assembly code.

```asm
        .file   "hello.c"
```

- `.file`: Self explanatory, the file name of the source file.


```asm
        .section        .rodata
.LC0:
        .string "Hello World"
```

- `.section`: Start of a section (could be, e.g., .section .rodata, .section .data)
- `.rodata`: Read-only data (e.g. global constants).
- `.LC0`: (Local Constants) Constants, such as strings.
- `.string`: Hopefully self explanatory.

```asm
        .text
```

- `.text`: Contains the code to be executed.

```asm
        .globl  main
        .type   main, @function
main:
.LFB0:
...
    ret
```

- `.globl` main: Defines the symbol `main` to be global (other compilation processes can access this symbol)
- `.type main, @function`: Tells the assembler that the symbol `main` is a function.
- `.LFB0`: (Local Function Begin) Start of a function.
- `ret`: Exists the function.

```asm
.LFE0:
        .size   main, .-main
        .ident  "GCC: (GNU) 15.2.1 20250813"
        .section        .note.GNU-stack,"",@progbits
```

- `.LFE0`: (Local Function End) End of a function.
- `.size`: Indicates the size of a function or symbol. Here, it says that main has size of `main` is the address from here (`.`) backwards (`-`) to the symbol `main`.
- `.ident`: Identification of how he program was compiled and when.
- The other section part can be ignored, basically means that this program can run in a stack that is non-executable (i.e. no code can be executed from the stack).

#### Binaries and flags

Directly running `gcc` on a `.c` file will create an output file, which is a **binary** file that runs the code.
You can change this name by simply using the `-o` flag, and specifying the output name.

You can add flags to change the behaviour of `gcc`, which in this case, `-Wall`, will print all compiler warnings.

#### Debugging

The holy mother of all debuggers, `gdb`, which stands for Gnu Debugger. Fun fact, see why we call bugs, well... bugs  [here](https://en.wikipedia.org/wiki/Bug_(engineering)#History)!


For debugging support, use the `-g` flag when invoking `gcc` (see what happens when you run `gcc -g -S hello.s`).

This is a pretty complicated debugger for modern standards. A good introduction can be found [here](https://www.gdbtutorial.com/) or [here](https://www.cs.cmu.edu/~gilpin/tutorial/).

### Notes on Exercise 2

Let us learn about linking! ([Source for image](https://www3.ntu.edu.sg/home/ehchua/programming/cpp/gcc_make.html)).

![Compilation process](gcc.png)

#### Pre-processing

**Input**: C code.
**Output**: Pre-processed C code.

So first we pre-process the `C` file to make sure that we expand the headers and macros via `cpp hello.c > hello.i`. This does not mean C++, but *C pre-processor*.

For example, part of the output include `extern int printf (const char *__restrict __format, ...);`, which comes from the `stdio.h` file.

#### Compilation

**Input**: Pre-processed C code.
**Output**: Assembly code.

We have seen this in the first exercise, which in reality takes the `hello.i` file instead of the `hello.c` file directly, but `gcc` handles this for us.

#### Assemble

**Input**: Assembly code.
**Output**: Machine code.

This converts assembly into machine code (code that can directly manipulate the CPU).
This is denoted by a `.o` file extension, which is short for `object`. This can be done by using the `as -o hello.o hello.s`.

#### Linker

You may need other bits of code that are spread around your computer (like libraries) to run the machine code.
This is the secret to the error in the second exercise, whereby `M_PI` comes from the C maths library!

Let us look at the code in the first exercise. If we simply run the liner via `ld -o hello -hello.o`, you will most likely get an error of the sort:

```sh
ld: warning: cannot find entry symbol _start; defaulting to 0000000000401000
ld: hello.o: in function `main':
hello.c:(.text+0xf): undefined reference to `puts'
```

That is because 1) The _start symbol is normally given by a system C library, and in the assembly code, we have this function call:

```asm
call    puts@PLT
```

The linker has no idea where in the world `puts` is, so we need to give it a library to get it from! For this to work properly, we need the `crt1.o`, `crti.o` and `crtn.o` libraries. To find out where they live:

```sh
user:~$ gcc -print-file-name=crt1.o     # entry point, calls __libc_start_main
/usr/lib/gcc/x86_64-pc-linux-gnu/15.2.1/../../../../lib/crt1.o

user:~$ gcc -print-file-name=crti.o     # .init/.fini section prologue
/usr/lib/gcc/x86_64-pc-linux-gnu/15.2.1/../../../../lib/crti.o

user:~$ gcc -print-file-name=crtn.o     # .init/.fini section epilogue

/usr/lib/gcc/x86_64-pc-linux-gnu/15.2.1/../../../../lib/crtn.o
```

So let us try to run:

```sh
user:~$ ld /usr/lib/crt1.o /usr/lib/crti.o /usr/lib/crtn.o hello.o
ld: /usr/lib/crt1.o: in function `_start':
(.text+0x21): undefined reference to `__libc_start_main'
ld: hello.o: in function `main':
hello.c:(.text+0xf): undefined reference to `puts'

```

More errors! Well, we need to include the standard C library, and the dynamic linker, which will allow the program to use shared libraries (i.e., outsource parts of the code) rather than having all the required code in the one program (e.g. outsourcing the `printf` code). To avoid dynamic libraries in this case, we would need a static C standard library implementation, which I do not believe exists by default in most Linux distributions.

```sh
user:~$ ld /usr/lib/crt1.o /usr/lib/crti.o /usr/lib/crtn.o -dynamic-linker /lib64/ld-linux-x86-64.so.2 -lc hello.o
user:~$ size a.out
   text    data     bss     dec     hex filename
    890     468       0    1358     54e a.out

user:~$ chmod a+x a.out
user:~$ sh a.out
Hello World

```

#### Note on VSCode

Developing C on VSCode can be tricky when using IntelliSence. Just make sure you include all necessary headers in your C program, such as `#include <math.h>`. You then should probably add the `-lm` to the IntelliSense args (see [https://code.visualstudio.com/docs/cpp/configure-intellisense#_option-2-edit-your-intellisense-configurations-through-the-ui](https://code.visualstudio.com/docs/cpp/configure-intellisense#_option-2-edit-your-intellisense-configurations-through-the-ui)).

### Notes on exercise 3

The annotated code:

```C
#include <stdio.h>

int main() {
  int i, j, *p, *q;
  i = 5; // variable i stored at address &i holds the value 5
  p = &i; // pointer p now stores address of i
  *p = 7; // the content at address &i (i.e. variable i) will now store the value 7 (this is called ponter dereferencing)
  j = 3; // variable j stored at address &j holds the value 3
  p = &j; // pointer p now stores the address of j (note that is it just 4 bytes above &i if the compiler chooses to put them side-by-side)
  q = p; // q holds the same address as p
  p = &i; // p once again contains address of i
  *q = 2; // q still holds address of j, so this will change the value of j
  return 0;
}
```

For a pointer `p`, `p` refers to the address `p` is storing, and `*p` to the contents stored within the address `p` holds.
We can be funky and refer to the address of `p` by using `&p`.

### Notes on exercise 4 and 5

In `C`, a pointer and an array are basically the same thing. For example, `char msg[] = "Hello World";` will reserve a region of memory of 12*1=12 bytes, and every byte is considered to be a new element in the array.

The variable `msg` now points to the beginning of this reserved region, so you can effectively move it up and down the array by incrementing *(msg + i) for some value integer `i`. If you want to permanently change point to the second element of the array, use `msg += 1`.

An integer takes 4 bytes, so to get the number of elements in an `int` array, simply divide the total size of the array by the size of an `int`.

### Notes on exercise 6

Only one hint: memory is forever, copies are temporary!

### Notes on exercise 7

In the first program, we are returning a local address of a variable that no longer exists! This is stupid because we are trying to read from a region of memory that either has nothing in it, or has been used by some other process and we are effectively reading memory we are not allowed to.

In the second program, we are effectively reserving 4 bytes of memory in the heap and assigning to pointer `p`. In some sense, we created a global array of one element.

We should free that memory afterwards, as we do not need it anymore.

### Notes on exercise 8

I will only talk about safety.

1. In this case, we will return to the function `f` after terminating `g`, so `x` still exists after `g` terminates. This is OK as long as `g` doesn't store the pointer in some place (e.g. global variable) after `f` finishes.

2. See exercise 7.

3. See exercise 7.

4. `g` receives a function pointer called `h` of type `int` that returns `int` – i.e., a pointer to a function taking an `int` and returning an `int`. Very safe, very good.
