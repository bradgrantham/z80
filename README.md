# SUZUKI PLAN - Z80 Emulator [![Build Status](https://travis-ci.org/suzukiplan/z80.svg?branch=master)](https://travis-ci.org/suzukiplan/z80)

The Z80 is an 8-bit CPU developed by Zilog corporation, released in 1976, and widely used in computers and game consoles in the 1980s.
It is not just a relic of the past, but continues to be used in embedded systems that require accuracy in processing execution time, such as real-time systems.

**SUZUKI PLAN - Z80 Emulator** is an emulator under development based on the following design guidelines to support the development of programs and/or emulators using Z80, 8080 or LR35902:

**(FOUR EASY GUIDELINES FOR EASILY)**

1. Make emulator implementation `EASY` & simple (Realized by providing single header: [z80.hpp](z80.hpp))
2. `EASILY` debugging the Z80 programs (Realized by having [dynamic disassemble feature](#dynamic-disassemble-for-debug))
3. Highly readable and `EASILY` customizable (priority for readability over performance)
4. Provide under the license that `EASY` to adopt in various programs ([MIT](LICENSE.txt))

> Since I do not have deep knowledge about Z80 myself, I'm implementing it with reference to the information on the following web sites:
>
> - [Z80 CPU User Manual - Zilog](https://www.zilog.com/manage_directlink.php?filepath=docs/z80/um0080&extn=.pdf)
> - [8 ビット CPU Z80](http://www.yamamo10.jp/yamamoto/comp/Z80/index.php) of [山本研究所](http://www.yamamo10.jp/yamamoto/index.html)
> - [Z80 Code Refference](http://mydocuments.g2.xrea.com/html/p6/z80ref.html) of [Bookworm's Library](http://mydocuments.g2.xrea.com/index.html)
> - [Zilog Z80 DAA Result Table](http://ver0.sakura.ne.jp/doc/daa.html) of [Version 0](http://ver0.sakura.ne.jp/)
> - [gbz80 — CPU opcode reference](https://rednex.github.io/rgbds/gbz80.7.html) of [RGBDS](https://github.com/rednex/rgbds)
> - [Gameboy CPU (LR35902) instruction set](https://pastraiser.com/cpu/gameboy/gameboy_opcodes.html)

## How to test

### Prerequests

You can test on the any 32bit/64bit platform/OS (UNIX, Linux, macOS...etc) but needs following middlewares:

- Git
- GNU Make
- clang
- clang-format

### Build (UNIX, Linux, macOS)

```
git clone https://github.com/suzukiplan/z80.git
cd z80
make
```

## Minimum usage

### 1. Include

```c++
#include "z80.hpp"
```

### 2. Implement MMU & access callback

```c++
class MMU
{
  public:
    unsigned char RAM[0x10000]; // 64KB memory (minimum)
    unsigned char IO[0x100]; // 256bytes port

    MMU()
    {
        memset(&RAM, 0, sizeof(RAM));
        memset(&IO, 0, sizeof(IO));
    }
};

// memory read request per 1 byte from CPU
unsigned char readByte(void* arg, unsigned short addr)
{
    // NOTE: implement switching procedure here if your MMU has bank switch feature
    return ((MMU*)arg)->RAM[addr];
}

// memory write request per 1 byte from CPU
void writeByte(void* arg, unsigned short addr, unsigned char value)
{
    // NOTE: implement switching procedure here if your MMU has bank switch feature
    ((MMU*)arg)->RAM[addr] = value;
}

// IN operand request from CPU
unsigned char inPort(void* arg, unsigned char port)
{
    return ((MMU*)arg)->IO[port];
}

// OUT operand request from CPU
void outPort(void* arg, unsigned char port, unsigned char value)
{
    ((MMU*)arg)->IO[port] = value;
}
```

### 3. Make Z80 or LR35902 instance

```c++
    MMU mmu;
    /**
     * readByte: callback of memory read request
     * writeByte: callback of memory write request
     * inPort: callback of input request
     * outPort: callback of output request
     * &mmu: 1st argument of the callbacks
     */
    Z80 z80(readByte, writeByte, inPort, outPort, &mmu);
```

> Specify NULL to inPort and outPort, if you'd like to use as LR35902:
>
> ```c++
>    Z80 gbz80(readByte, writeByte, NULL, NULL, &mmu);
> ```
>
> _LR35902: known as CPU for Nintendo GameBoy series._

### 4. Execute

```c++
    // when executing about 1234Hz
    int actualExecuteClocks = z80.execute(1234);
```

### 5. Generate interrupt

#### IRQ; Interrupt Request

```c++
    z80.generateIRQ(vector);
```

#### NMI; Non Maskable Interrupt

```c++
    z80.generateNMI(address);
```

## Optional features

### Dynamic disassemble (for debug)

You can acquire the debug messages with `setDebugMessage`.
Debug message contains dynamic disassembly results step by step.

```c++
    z80.setDebugMessage([](void* arg, const char* message) -> void {
        time_t t1 = time(NULL);
        struct tm* t2 = localtime(&t1);
        printf("%04d.%02d.%02d %02d:%02d:%02d %s\n",
               t2->tm_year + 1900, t2->tm_mon, t2->tm_mday, t2->tm_hour,
               t2->tm_min, t2->tm_sec, message);
    });
```

### Use break point

If you want to execute processing just before executing an instruction of specific program counter value _(in this ex: \$008E)_, you can set a breakpoint as follows:

```c++
    z80.addBreakPoint(0x008E, [](void* arg) -> void {
        printf("Detect break point! (PUSH ENTER TO CONTINUE)");
        char buf[80];
        fgets(buf, sizeof(buf), stdin);
    });
```

- `addBreakPoint` can set multiple breakpoints.
- call `removeBreakPoint` or `removeAllBreakPoints` if you want to remove the break point(s).

### Use break operand

If you want to execute processing just before executing an instruction of specific operand number _(in this ex: \$00; NOP)_, you can set a breakpoint as follows:

```c++
    z80.addBreakOperand(0x00, [](void* arg) -> void {
        printf("Detect break operand! (PUSH ENTER TO CONTINUE)");
        char buf[80];
        fgets(buf, sizeof(buf), stdin);
    });
```

- `addBreakOperand` can set multiple breakpoints.
- call `removeBreakOperand` or `removeAllBreakOperands` if you want to remove the break operand(s).

### Detect clock consuming

If you want to implement stricter synchronization, you can capture the CPU clock consumption timing as follows:

```c++
    z80.setConsumeClockCallback([](void* arg, int clock) -> void {
        printf("consumed: %dHz\n", clock);
    });
```

- call `setConsumeClockCallback(NULL)` if you want to remove the detector.

> With this callback, the CPU cycle (clock) can be synchronized in units of 3 to 4 Hz, and while the execution of a single Z80 instruction requires approximately 10 to 20 Hz of CPU cycle (time), the SUZUKI PLAN - Z80 Emulator can synchronize the CPU cycle (time) for fetch, execution, write back, etc. However, the SUZUKI PLAN - Z80 Emulator can synchronize fetches, executions, writes, backs, etc. in smaller units. This makes it easy to implement severe timing emulation.

### If implement quick save/load

Save the member variable `reg` when quick saving:

```c++
    fwrite(&z80.reg, sizeof(z80.reg), 1, fp);
```

Extract to the member variable `reg` when quick loading:

```c++
    fread(&z80.reg, sizeof(z80.reg), 1, fp);
```

### Handling of CALL instructions

The occurrence of the branches by the CALL instructions can be captured by the CallHandler.
CallHandler will be called back immediately **after** a branch by a CALL instruction occurs.

> CallHandle will called back after the return address is stacked in the RAM.

```c++
    z80.addCallHandler([](void* arg) -> void {
        printf("Executed a CALL instruction:\n");
        printf("- Branched to: $%04X\n", ((Z80*)arg)->reg.PC);
        unsigned short sp = ((Z80*)arg)->reg.SP;
        unsigned short returnAddr = ((Z80*)arg)->readByte(sp + 1);
        returnAddr <<= 8;
        returnAddr |= ((Z80*)arg)->readByte(sp);
        printf("- Return to: $%04X\n", returnAddr);
    });
```

- `addCallHandler` can set multiple CallHandlers.
- call `removeCallHandler` or `removeAllCallHandlers` if you want to remove the CallHandler(s).
- CallHandler also catches branches caused by interrupts.
- In the case of a condition-specified branch instruction, only the case where the branch is executed is callbacked.

### Handling of RET instructions

The occurrence of the branches by the RET instructions can be captured by the ReturnHandler.
ReturnHandler will be called back immediately **before** a branch by a RET instruction occurs.

> ReturnHandle will called back while the return address is stacked in the RAM.

```c++
    z80.addReturnHandler([](void* arg) -> void {
        printf("Detected a RET instruction:\n");
        printf("- Branch from: $%04X\n", ((Z80*)arg)->reg.PC);
        unsigned short sp = ((Z80*)arg)->reg.SP;
        unsigned short returnAddr = ((Z80*)arg)->readByte(sp + 1);
        returnAddr <<= 8;
        returnAddr |= ((Z80*)arg)->readByte(sp);
        printf("- Return to: $%04X\n", returnAddr);
    });
```

- `addReturnHandler` can set multiple ReturnHandlers.
- call `removeReturnHandler` or `removeAllReturnHandlers` if you want to remove the ReturnHandler(s).
- In the case of a condition-specified branch instruction, only the case where the branch is executed is callbacked.

## License

[MIT](LICENSE.txt)
