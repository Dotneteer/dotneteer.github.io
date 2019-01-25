---
layout: post
title:  "ZX Spectrum IDE — Part #5: Implementing Z80 instructions (1)"
date:   2018-02-01 13:20:00 +0100
categories: "ZX-Spectrum"
abstract: >- 
  The CPU has more than 1300 instructions, and thus Z80Cpu should take care each of them. In this post, you will learn the implementation details behind a few Z80 instructions.
---

In the [previous article](/zx-spectrum/2018/01/26/zxspectrum-part-4.html), you learned the internal architecture of the `Z80Cpu` class that implements the CPU emulation in SpectNetIde. The CPU has more than 1300 instructions, and thus `Z80Cpu` should take care each of them. In this post, you will learn the implementation details behind a few Z80 instructions.

### Documentation and Tests

When designing the emulation architecture, I took care building it to be easily testable. The current [SpectNetIde](https://github.com/Dotneteer/spectnetide) project tests each instruction separately; most instructions have more than one unit test cases. In the next article, I will show you how I implemented those tests.

Besides testing, I intended to create the source code so that you can immediately understand the specification of a particular instruction—without jumping to the Z80 reference documentation.

I added the reference documentation to the XML comments of each instruction methods, as this sample (__ADD A,B__) shows:

```csharp
/// <summary>
///     add a,b
/// </summary>
/// <remarks>
///     The contents of B are added to the contents of A, and the result is
///     stored in A.
///     S is set if result is negative; otherwise, it is reset.
///     Z is set if result is 0; otherwise, it is reset.
///     H is set if carry from bit 3; otherwise, it is reset.
///     P/V is set if overflow; otherwise, it is reset.
///     N is reset.
///     C is set if carry from bit 7; otherwise, it is reset.
///     =================================
///     | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0x80
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void AddA_B()
{
    var src = _registers.B;
    _registers.F = s_AdcFlags[_registers.A * 0x100 + src];
    _registers.A += src;
}
```

The documentation starts with a short description of the operation. A part of Z80 instructions does not modify the flags at all, while others do. After the explanation, I treat how a specific instruction handles the flags.
The __T-States__ value indicates the number of clock cycles the instruction takes to carry out. The contention breakdown entry describes how a particular instruction behaves on ZX Spectrum in a contended situation. Later, in the article that treats memory and I/O contention, I will tell you how to decode the content of that field. Right now, just ignore it.
Just for a short recap, here is the list of Z80 flags:

Flag | Description
--------|---
__C__&nbsp;(Bit&nbsp;0) | __Carry flag__. It is set or cleared depending on the operation is performed. For ALU operations, it signs carry (e.g., __ADD__) or borrow (e.g., __SUB__). For bit shift and rotate operations, it stores the least/most significant bit after an operation. For the logical instructions __AND__, __OR__, and __XOR__, the Carry flag is reset.
__N__&nbsp;(Bit&nbsp;1) | __Add/Subtract flag__. This flag is used by the _Decimal Adjust Accumulator_ instruction (__DAA__) to distinguish between the __ADD__ and __SUB__ instructions. For __ADD__ instructions, __N__ is cleared to 0. For __SUB__ instructions, __N__ is set to 1.
__P/V__&nbsp;(Bit&nbsp;2) | __Parity/Overflow flag__. This flag is set to a specific state depending on the operation being performed. For arithmetic operations, this flag indicates an overflow condition when the result in the __Accumulator__ is greater than the maximum possible number (+127) or is less than the minimum possible number (–128). This overflow condition is determined by examining the sign bits of the operands.
__H__&nbsp;(Bit&nbsp;4) | __Half Carry flag__. This flag is set or cleared depending on the carry and borrow status between bits 3 and 4 of an 8-bit arithmetic operation. This flag is used by the _Decimal Adjust Accumulator_ (__DAA__) instruction to correct the result of a packed BCD add or subtract operation.
__Z__&nbsp;(Bit&nbsp;6) | __Zero flag__. It is set if the result generated by the execution of certain instructions is 0; otherwise, it is reset.
__S__&nbsp;(Bit&nbsp;7) | __Sign flag__. It stores the state of the most-significant bit of the Accumulator (bit 7).

_Note: There are two undocumented flags, Bit 3 and Bit 5 of the __F__ register. These flags cannot be read directly. They store the 3rd and 5th bit of the result for every operation that changes any flag. In the emulator, I use the names __R3__ and __R5__ for these flags._

You probably remember that the `Z80Cpu` class uses jump tables to invoke actions associated with operation codes. In this post, I will show end explain the methods behind these actions.

### Simple Instructions

Many Z80 instructions are simple. They work with registers, load them from the memory, or store them. Here, I show a few of them.

#### NOP

The simplest is the __NOP__ (_No Operation_) instruction. The CPU executes its __M1__ cycle without any further processing. Thus, the __NOP__ instruction even does not have a dedicated action method. The `ExecuteCpuCycle()` method does this job with these lines:

```csharp
...
var opCode = ReadMemory(_registers.PC);
ClockP3();
_registers.PC++;
RefreshMemory();
...
```

#### 8-Bit Register-To-Register Load

The Z80 CPU has 49 operations to move data from one of the seven 8-bit registers to another one. This example shows the __LD B,C__ operation, which could not be implemented simpler:

```csharp
/// <summary>
///     "ld b,c" operation
/// </summary>
/// <remarks>
///     The contents of C are loaded to B.
///     =================================
///     | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 1 | 0x41
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void LdB_C()
{
    _registers.B = _registers.C;
}
```

All remaining 8-bit-register-to-8-bit-register operations use the same approach with a single line transfer code.

#### Loading Value to an 8-Bit Register

The CPU has instructions to move 8-bit literal values from the code to an 8-bit register, such as the code of the __LD E,N__ operation shows:

```csharp
/// <summary>
///     "ld e,N" operation
/// </summary>
/// <remarks>
///     The 8-bit integer N is loaded to E.
///     =================================
///     | 0 | 0 | 0 | 1 | 1 | 1 | 1 | 0 | 0x1E
///     =================================
///     |            8-bit              |
///     =================================
///     T-States: 4, 3 (7)
///     Contention breakdown: pc:4,pc+1:3
/// </remarks>
private void LdEN()
{
    _registers.E = ReadMemory(_registers.PC);
    ClockP3();
    _registers.PC++;
}
```

Here, __N__ is an 8-bit value that follows the opcode in the memory. By the time the code invokes the `LdEN()` method, __PC__ points to __N__ in the memory.

Because a memory read operation takes 3 T-states, the `ClockP3()` method adjusts the `Tacts` counter.

_Note: At first sight, it does not matter where we put `ClockP3()` in the code because anywhere we put it, it always increases `Tacts` with 3. Well, it is not so. Because of memory contention, we need to add it after the memory read operation. The reason behind this approach is that the `ReadMemory()` operation may adjust the counter. The amount of this adjustment is a function of two inputs: the current `Tacts` value, and the memory address, respectively. Moving `ClockP3()` before the `ReadMemory()` call might result a different clock adjustment.  You will read more details later in the article about memory and I/O contention._

#### Loading Value to a 16-Bit Register

The 16-bit value loading operation follows the same logic as its 8-bit version pair. This code shows the internals of the __LD DE,NN__ instruction, where __NN__ is a 16-bit value stored in LSB/MSB order right after the opcode.

```csharp
/// <summary>
///     "ld de,NN" operation
/// </summary>
/// <remarks>
///     The 16-bit integer value is loaded to the DE register pair.
///     =================================
///     | 0 | 0 | 0 | 1 | 0 | 0 | 0 | 1 | 0x11
///     =================================
///     |             N Low             |
///     =================================
///     |             N High            |
///     =================================
///     T-States: 4, 3, 3 (10)
///     Contention breakdown: pc:4,pc+1:3,pc+2:3
/// </remarks>
private void LdDENN()
{
     _registers.E = ReadMemory(_registers.PC);
    ClockP3();
    _registers.PC++;
    _registers.D = ReadMemory(_registers.PC);
    ClockP3();
    _registers.PC++;
}
```

You can see that the code carries out two read operations after each other. The method stores the result of the first read in the E register (LSB), the second in D. Similarly to the previous operation the two `ClockP3()` calls cannot be changed to a single `ClockP6()`, as it would not correctly handle memory contention.

#### Loading an 8-Bit Register from Memory

The following code executes the __LD A,(BC)__ operation. It works exactly as you imagine. Nonetheless, the code sets up the value of the internal __WZ__ register:

```csharp
/// <summary>
///     "ld a,(bc)" operation
/// </summary>
/// <remarks>
///     The contents of the memory location specified by BC are loaded to A.
///     =================================
///     | 0 | 0 | 0 | 0 | 1 | 0 | 1 | 0 | 0x0A
///     =================================
///     T-States: 4, 3 (7)
///     Contention breakdown: pc:4,bc:3
/// </remarks>
private void LdABCi()
{
    _registers.WZ = (ushort) (_registers.BC + 1);
    _registers.A = ReadMemory(_registers.BC);
    ClockP3();
}
```

You may ask, why we set the value of an internal register if it’s not available from program code. Well, the Z80 CPU has some officially undocumented behavior (e.g., BIT instruction), where the value of __WZ__ influences how the undocumented __R3__ and __R5__ flags are calculated. Nonetheless, the interesting thing is that though we read from the address pointed by __BC__, __WZ__ is set to __BC+1__. I won’t explain why, it would take long. It is the internal behavior of the Z80 CPU.

#### Loading a 16-Bit Register from Memory

As the code of the __LD HL,(NN)__ instruction shows, we need four read operations to get the value of a 16-bit register from an actual memory address:

```csharp
/// <summary>
///     "ld hl,(NN)" operation
/// </summary>
/// <remarks>
///     The contents of memory address (NN) are loaded to the
///     low-order portion of HL (L), and the contents of the next
///     highest memory address (NN + 1) are loaded to the high-order
///     portion of HL (H).
///     =================================
///     | 0 | 0 | 1 | 0 | 1 | 0 | 1 | 0 | 0x2A
///     =================================
///     |           8-bit L             |
///     =================================
///     |           8-bit H             |
///     =================================
///     T-States: 4, 3, 3, 3, 3 (16)
///     Contention breakdown: pc:4,pc+1:3,pc+2:3,nn:3,nn+1:3
/// </remarks>
private void LdHLNNi()
{
    ushort adr = ReadMemory(_registers.PC);
    ClockP3();
    _registers.PC++;
    adr += (ushort) (ReadMemory(_registers.PC) << 8);
    ClockP3();
    _registers.PC++;
    _registers.WZ = (ushort) (adr + 1);
    ushort val = ReadMemory(adr);
    ClockP3();
    val += (ushort) (ReadMemory(_registers.WZ) << 8);
    ClockP3();
    _registers.HL = val;
}
```

The first two reads collect the address of the memory; the subsequent two reads obtain the LSB and MSB of the register’s new value from the memory.

#### Storing an 8-Bit Register’s Value into Memory

The Z80 has instruction that writes into a 16-bit memory address, such as __LD (NN),A__. This instruction first obtains the memory address, and then stores the value of __A__ to that address:

```csharp
/// <summary>
///     "ld a,(NN)" operation
/// </summary>
/// <remarks>
///     The contents of A are loaded to the memory address specified by
///     the operand NN
///     =================================
///     | 0 | 0 | 1 | 1 | 1 | 0 | 1 | 0 | 0x32
///     =================================
///     |           8-bit L             |
///     =================================
///     |           8-bit H             |
///     =================================
///     T-States: 4, 3, 3, 3 (13)
///     Contention breakdown: pc:4,pc+1:3,pc+2:3,nn:3
/// </remarks>
private void LdNNA()
{
    var l = ReadMemory(_registers.PC);
    ClockP3();
    _registers.PC++;
    var addr = (ushort) ((ReadMemory(_registers.PC) << 8) | l);
    ClockP3();
    _registers.PC++;
    _registers.WZ = (ushort) (((addr + 1) & 0xFF) + (_registers.A << 8));
    WriteMemory(addr, _registers.A);
    _registers.WZh = _registers.A;
    ClockP3();
}
```

The code is straightforward, except the snippet that sets the value of __WZ__. As you can see, it happens in two steps (the data bus can handle eight bits in a single transfer). __WZh__ signifies the upper eight bits of __WZ__.

_Note: From now on, I will explain the reason for using __WZ__ only when I intend to point out to something significant. Otherwise, I just ignore the explanation._

#### Storing an 8-Bit Register’s Value into Memory

I guess the implementation of the __LD (NN),HL__ instruction is straightforward:

```csharp
/// <summary>
///     "ld (NN),hl" operation
/// </summary>
/// <remarks>
///     The contents of the low-order portion of HL (L) are loaded to memory
///     address (NN), and the contents of the high-order portion of HL (H)
///     are loaded to the next highest memory address(NN + 1).
///     =================================
///     | 0 | 0 | 1 | 0 | 0 | 0 | 1 | 0 | 0x22
///     =================================
///     |           8-bit L             |
///     =================================
///     |           8-bit H             |
///     =================================
///     T-States: 4, 3, 3, 3, 3 (16)
///     Contention breakdown: pc:4,pc+1:3,pc+2:3,nn:3,nn+1:3
/// </remarks>
private void LdNNiHL()
{
    var l = ReadMemory(_registers.PC);
    ClockP3();
    _registers.PC++;
    var addr = (ushort) ((ReadMemory(_registers.PC) << 8) | l);
    ClockP3();
    _registers.PC++;
    _registers.WZ = (ushort) (addr + 1);
    WriteMemory(addr, _registers.L);
    ClockP3();
    WriteMemory(_registers.WZ, _registers.H);
    ClockP3();
}
```

### More about Reading and Writing Memory

The `Z80Cpu` class outsources memory handling functionality to an abstract `IMemoryDevice`:

```csharp
public  partial class Z80Cpu: IZ80Cpu
{
    // ...
    private readonly IMemoryDevice _memoryDevice;
    // ...
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public byte ReadMemory(ushort addr) => 
        _memoryDevice.Read(addr);

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void WriteMemory(ushort addr, byte value) => 
        _memoryDevice.Write(addr, value);
}
```

This is an essential implementation detail. Doing so, the emulator can easily handle features such as paging, banking, handling ROM, and so on. This indirection makes testing easier, too.

### ALU Operations

Implementing ALU operations seems to be evident at first sight. How could be adding or subtracting two 8-bit or 16-bit value difficult? When emulating these operations, the challenge is to manage flag changes efficiently, for these instructions change flags profoundly.

#### Incrementing a 16-Bit Register’s Value

Incrementing a 16-bit register keeps all flags unaffected. Thus, as the code of __INC HL__ shows, the implementation is as simple as you expect:

```csharp
/// <summary>
///     "inc hl" operation
/// </summary>
/// <remarks>
///     The contents of register pair HL are incremented.
///     =================================
///     | 0 | 0 | 1 | 0 | 0 | 0 | 1 | 1 | 0x23
///     =================================
///     T-States: 4, 2 (6)
///     Contention breakdown: pc:6
/// </remarks>
private void IncHL()
{
    _registers.HL++;
    ClockP2();
}
```

#### Incrementing an 8-Bit Register’s value

Unlike a 16-bit increment instruction, an 8-bit operation changes flag values. The code of __INC D__ shows how it goes:

```csharp
/// <summary>
///     "inc d" operation
/// </summary>
/// <remarks>
///     Register D is incremented.
///     S is set if result is negative; otherwise, it is reset.
///     Z is set if result is 0; otherwise, it is reset.
///     H is set if carry from bit 3; otherwise, it is reset.
///     P/V is set if r was 7Fh before operation; otherwise, it is reset.
///     N is reset.
///     C is not affected.
///     =================================
///     | 0 | 0 | 0 | 1 | 0 | 1 | 0 | 0 | 0x14
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void IncD()
{
    _registers.F = (byte) (s_IncOpFlags[_registers.D++] 
        | (_registers.F & FlagsSetMask.C));
}
```

According to the comment, this instruction may change seven flags out of eight (let’s not forget about the two undocumented flags, __R3__ and __R5__). It keeps unaffected only __C__. For performance reason, I do not set these flags individually, but use a predefined table, `s_IncOpFlags`, to obtain the value of flags after the increment operation. Observe how the implementation keeps the Carry flag untouched.

Because we work with eight bits of data, we can easily pre-calculate the flag values. The implementation contains ALU helper tables within the `Z80Cpu` class. When the CPU instance is constructed, the `InitializeAluTables()` method prepares these tables. This is the code snippet that calculates the contents of `s_IncOpFlags`:

```csharp
...
s_IncOpFlags = new byte[0x100];
for (var b = 0; b < 0x100; b++)
{
    var oldVal = (byte) b;
    var newVal = (byte) (oldVal + 1);
    var flags =
    // C is unaffected, we keep it false
    (newVal & FlagsSetMask.R3) |
    (newVal & FlagsSetMask.R5) |
    ((newVal & 0x80) != 0 ? FlagsSetMask.S : 0) |
        (newVal == 0 ? FlagsSetMask.Z : 0) |
        ((oldVal & 0x0F) == 0x0F ? FlagsSetMask.H : 0) |
        (oldVal == 0x7F ? FlagsSetMask.PV : 0);
    // N is false
    s_IncOpFlags[b] = (byte) flags;
}
...
```

_Note: The `FlagSetMask` class contains constant values to mask out a particular flag from the __F__ register. There’s another class, `FlagResetMask` with constant values to reset the specific flag while keeping others from __F__.

#### Adding Two 8-Bit Registers

Using ALU helper tables is a good technique, but as you can see from the implementation details of the __ADD A,H__ instruction, the price of the performance is increased usage of memory:

```csharp
/// <summary>
///     add a,h
/// </summary>
/// <remarks>
///     The contents of H are added to the contents of A, and the result is
///     stored in A.
///     S is set if result is negative; otherwise, it is reset.
///     Z is set if result is 0; otherwise, it is reset.
///     H is set if carry from bit 3; otherwise, it is reset.
///     P/V is set if overflow; otherwise, it is reset.
///     N is reset.
///     C is set if carry from bit 7; otherwise, it is reset.
///     =================================
///     | 1 | 0 | 0 | 0 | 0 | 1 | 0 | 0 | 0x84
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void AddA_H()
{
    var src = _registers.H;
    _registers.F = s_AdcFlags[_registers.A * 0x100 + src];
    _registers.A += src;
}
```

Here, the `s_AdcFlags` table contains 2 * 256 *256 entries: we need to combine 256 different value of __A__ with 256 potential values of __H__. Besides, we have two additions, __ADD__, which ignores the carry flag, and __ADC__, which utilizes carry. This is how I calculate `s_AdcFlags` entries:

```csharp
...
s_AdcFlags = new byte[0x20000];
for (var C = 0; C < 2; C++)
{
    for (var X = 0; X < 0x100; X++)
    {
        for (var Y = 0; Y < 0x100; Y++)
        {
            var res = (ushort) (X + Y + C);
            var flags = 0;
            if ((res & 0xFF) == 0) flags |= FlagsSetMask.Z;
            flags |= res & (FlagsSetMask.R3 | FlagsSetMask.R5 | FlagsSetMask.S);
            if (res >= 0x100) flags |= FlagsSetMask.C;
            if ((((X & 0x0F) + (Y & 0x0F) + C) & 0x10) != 0) flags |= FlagsSetMask.H;
            var ri = (sbyte) X + (sbyte) Y + C;
            if (ri >= 0x80 || ri <= -0x81) flags |= FlagsSetMask.PV;
            s_AdcFlags[C * 0x10000 + X * 0x100 + Y] = (byte) flags;
        }
    }
}
...
```

For the sake of completeness, here is the code of __ADC A,E__. You can observe how the carry flag is weaved into the operation:

```csharp
/// <summary>
///     adc a,e
/// </summary>
/// <remarks>
///     The contents of E and the C flag are added to the contents of A,
///     and the result is stored in A.
///     S is set if result is negative; otherwise, it is reset.
///     Z is set if result is 0; otherwise, it is reset.
///     H is set if carry from bit 3; otherwise, it is reset.
///     P/V is set if overflow; otherwise, it is reset.
///     N is reset.
///     C is set if carry from bit 7; otherwise, it is reset.
///     =================================
///     | 1 | 0 | 0 | 0 | 1 | 0 | 1 | 1 | 0x8B
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void AdcA_E()
{
    var src = _registers.E;
    var carry = (_registers.F & FlagsSetMask.C) == 0 ? 0 : 1;
    _registers.F = s_AdcFlags[carry * 0x10000 + _registers.A * 0x100 + src];
    _registers.A += (byte) (src + carry);
}
```

#### Adding Two 16-Bit Registers

Today, when we have gigabytes of memory in our computers (and even in mobile devices), storing 128 Kbytes of pre-calculated data seems to be a good tradeoff for the performance gain. However, when we execute ALU operations with two 16-bit numbers, we had to store 8 Gbytes of data for such a helper table. Apparently, it would not be a viable tradeoff. We need to calculate the flag values during run time. To demonstrate this, here is the code behind the __ADC HL,DE__ operation:

```csharp
/// <summary>
/// "ADC HL,QQ" operation
/// </summary>
/// <remarks>
/// 
/// The contents of register pair QQ are added with the Carry flag
/// to the contents of HL, and the result is stored in HL.
/// 
/// S is set if result is negative; otherwise, it is reset.
/// Z is set if result is 0; otherwise, it is reset.
/// H is set if carry from bit 11; otherwise, it is reset.
/// P/V is set if overflow; otherwise, it is reset.
/// N is reset.
/// C is set if carry from bit 15; otherwise, it is reset.
///
/// =================================
/// | 1 | 1 | 1 | 0 | 1 | 1 | 0 | 1 | ED
/// =================================
/// | 0 | 1 | Q | Q | 1 | 0 | 1 | 0 |
/// =================================
/// QQ: 00=BC, 01=DE, 10=HL, 11=SP
/// T-States: 4, 4, 4, 3 (15)
/// Contention breakdown: pc:4,pc+1:11
/// </remarks>
private void ADCHL_QQ()
{
    _registers.WZ = (ushort)(_registers.HL + 1);
    var cfVal = _registers.CFlag ? 1 : 0;
    var qq = (Reg16Index)((_opCode & 0x30) >> 4);
    var flags = (byte)((((_registers.HL & 0x0FFF) + (_registers[qq] & 0x0FFF)
        + (_registers.F & FlagsSetMask.C)) >> 8) & FlagsSetMask.H);
    var adcVal = (uint)((_registers.HL & 0xFFFF) + (_registers[qq] & 0xFFFF) 
        + cfVal);
    if ((adcVal & 0x10000) != 0)
    {
        flags |= FlagsSetMask.C;
    }
    if ((adcVal & 0xFFFF) == 0)
    {
        flags |= FlagsSetMask.Z;
    }
    var signedAdc = (short)_registers.HL + (short)_registers[qq] + cfVal;
    if (signedAdc < -0x8000 || signedAdc >= 0x8000)
    {
        flags |= FlagsSetMask.PV;
    }
    _registers.HL = (ushort)adcVal;
    _registers.F = (byte)(flags | (_registers.H & (FlagsSetMask.S | FlagsSetMask.R3 
        | FlagsSetMask.R5)));
    ClockP7();
}
```

The first thing you observe is that the method’s name is not `ADCHL_DE`, but `ADCHL_QQ`. It is not misnaming. __QQ__ represent that this operation works with any of the __BC__, __DE__, __HL__, and __SP__ registers; this method implements all the __ADC HL,BC__, __ADC HL,DE__, __ADC HL,HL__, and __ADC HL,SP__ instructions.

These are extended operations with the __$ED__ opcode prefix. Bit 4 and 5 of the second opcode byte names the second operand register, the value of which is queried with this code line:

```csharp
var qq = (Reg16Index)((_opCode & 0x30) >> 4);
```

The `Registers` class provides an indexer property to access to 16-bit registers (and another indexer to get or set 8-bit registers). The `_registers[qq]` expression gets the value of the register specified by the second opcode byte.

#### Bit Test Instructions

The __BIT N,Q__ instruction, which tests if the _Nth_ bit of the __Q__ 8-bit register is set, used opcode indirection for both __N__ and __Q__:

```csharp
/// <summary>
/// "BIT N,Q" operation
/// </summary>
/// <remarks>
/// 
/// This instruction tests bit N in register Q and sets the Z 
/// flag accordingly.
/// 
/// S Set if N = 7 and tested bit is set.
/// Z is set if specified bit is 0; otherwise, it is reset.
/// H is set.
/// P/V is Set just like ZF flag.
/// N is reset.
/// C is not affected.
/// 
/// =================================
/// | 1 | 1 | 0 | 0 | 1 | 0 | 1 | 1 | CB prefix
/// =================================
/// | 0 | 1 | N | N | N | Q | Q | Q |
/// =================================
/// Q: 000=B, 001=C, 010=D, 011=E
///    100=H, 101=L, 110=N/A, 111=A
/// T-States: 4, 4 (8)
/// Contention breakdown: pc:4,pc+1:4
/// </remarks>
private void BITN_Q()
{
    var q = (Reg8Index) (_opCode & 0x07);
    var n = (byte) ((_opCode & 0x38) >> 3);
    var srcVal = _registers[q];
    var testVal = srcVal & (1 << n);
    var flags = FlagsSetMask.H
        | (_registers.F & FlagsSetMask.C)
        | (srcVal & (FlagsSetMask.R3 | FlagsSetMask.R5));
    if (testVal == 0)
    {
        flags |= FlagsSetMask.Z | FlagsSetMask.PV;
    }
    if (n == 7 && testVal != 0)
    {
        flags |= FlagsSetMask.S;
    }
    _registers.F = (byte)flags;
}
```

As the code (and its comment) shows, Bit 0, 1, and 2 name __Q__, Bit 3, 4, and 5 specify __N__.

The `BITN_Q` method itself carries out all the 64 bit-test operations that you can execute with the __$CB__ prefix. Instead of calculating flag values run time, I could also create a helper table with 8 * 256 entries (8 entries for __N__, 256 entries for each 8-bit values) to accelerate the calculation. Well, this method is a good candidate for such performance refactoring.

I could use a single method body to implement the 8-bit-register-to-8-bit register load operations. However, I created 64 separate methods. I did it because I opted to avoid two indirections (getting the value of the source register and setting the value of the destination register) for the sake of performance. So, such a transfer operation (e.g. __LD D,B__) is so simple:

```csharp
_registers.D = _registers.B;
```

#### Shift and Rotate Instructions

I used helper tables for shift and rotate instructions with pre-calculated flag values. Here is the implementation of the __SLA D__ operation:

```csharp
/// <summary>
/// "sla d" operation
/// </summary>
/// <remarks>
/// 
/// An arithmetic shift left 1 bit position is performed on the 
/// contents of register D. The contents of bit 7 are copied to 
/// the Carry flag.
/// 
/// S, Z, P/V are not affected.
/// H, N are reset.
/// C is data from bit 7 of the original register value.
/// 
/// =================================
/// | 1 | 1 | 0 | 0 | 1 | 0 | 1 | 1 | 0xCB
/// =================================
/// | 0 | 0 | 1 | 0 | 0 | 0 | 1 | 0 | 0x22
/// =================================
/// T-States: 4, 4 (8)
/// Contention breakdown: pc:4,pc+1:4
/// </remarks>
private void SLA_D()
{
    int slaVal = _registers.D;
    _registers.F = s_RlCarry0Flags[(byte)slaVal];
    _registers.D = (byte)(slaVal << 1);
}
```

The __RR L__ operation copies the previous carry flag into bit 7. In this implementation, I use two helper tables, according to the value of the carry:

```csharp
/// <summary>
/// "rr l" operation
/// </summary>
/// <remarks>
/// 
/// The contents of register L are rotated right 1 bit position 
/// through the Carry flag. The contents of bit 0 are copied to the 
/// Carry flag and the previous contents of the Carry flag are 
/// copied to bit 7.
/// 
/// S, Z, P/V are not affected.
/// H, N are reset.
/// C is data from bit 0 of the original register value.
/// 
/// =================================
/// | 1 | 1 | 0 | 0 | 1 | 0 | 1 | 1 | 0xCB
/// =================================
/// | 0 | 0 | 0 | 1 | 1 | 1 | 0 | 1 | 0x1D
/// =================================
/// T-States: 4, 4 (8)
/// Contention breakdown: pc:4,pc+1:4
/// </remarks>
private void RR_L()
{
    int rrVal = _registers.L;

    if (_registers.CFlag)
    {
        _registers.F = s_RrCarry1Flags[rrVal];
        _registers.L = (byte)((rrVal >> 1) + 0x80);
    }
    else
    {
        _registers.F = s_RrCarry0Flags[rrVal];
        _registers.L = (byte)(rrVal >> 1);
    }
}
```

#### Logical operations

The Z80 CPU provides logical operations between __A__ and the other 8-bit registers, such as __OR__, __AND__, __XOR__. Their implementation uses the same helper table, `s_AluOpFlags`, as the implementation of __OR C__ and __AND C__ shows:

```csharp
/// <summary>
///     or c
/// </summary>
/// <remarks>
///     A logical OR operation is performed between C and the byte contained in A;
///     the result is stored in the Accumulator.
///     S is set if result is negative; otherwise, it is reset.
///     Z is set if result is 0; otherwise, it is reset.
///     H is reset.
///     P/V is reset if overflow; otherwise, it is reset.
///     N is reset.
///     C is reset.
///     =================================
///     | 1 | 0 | 1 | 1 | 0 | 0 | 0 | 1 | 0xB1
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void OrC()
{
    var src = _registers.C;
    _registers.A |= src;
    _registers.F = s_AluLogOpFlags[_registers.A];
}

/// <summary>
///     and c
/// </summary>
/// <remarks>
///     A logical AND operation is performed between C and the byte contained in A;
///     the result is stored in the Accumulator.
///     S is set if result is negative; otherwise, it is reset.
///     Z is set if result is 0; otherwise, it is reset.
///     H is set.
///     P/V is reset if overflow; otherwise, it is reset.
///     N is reset.
///     C is reset.
///     =================================
///     | 1 | 0 | 1 | 0 | 0 | 0 | 0 | 1 | 0xA1
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void AndC()
{
    var src = _registers.C;
    _registers.A &= src;
    _registers.F = (byte) (s_AluLogOpFlags[_registers.A] | FlagsSetMask.H);
}
```

#### The DAA Instruction

Believe it or not, the DAA instruction is probably the most complicated Z80 instruction, though its implementation does not reflect this fact:

```csharp
/// <summary>
///     "daa" operation
/// </summary>
/// <remarks>
///     This instruction conditionally adjusts A for BCD addition
///     and subtraction operations. For addition(ADD, ADC, INC) or
///     subtraction(SUB, SBC, DEC, NEG), the following table indicates
///     the operation being performed:
///     ====================================================
///     |Oper.|C before|Upper|H before|Lower|Number|C after|
///     |     |DAA     |Digit|Daa     |Digit|Added |Daa    |
///     ====================================================
///     | ADD |   0    | 9-0 |   0    | 0-9 |  00  |   0   |
///     |     |   0    | 0-8 |   0    | A-F |  06  |   0   |
///     |     |   0    | 0-9 |   1    | 0-3 |  06  |   0   |
///     |     |   0    | A-F |   0    | 0-9 |  60  |   1   |
///     ----------------------------------------------------
///     | ADC |   0    | 9-F |   0    | A-F |  66  |   1   |
///     ----------------------------------------------------
///     | INC |   0    | A-F |   1    | 0-3 |  66  |   1   |
///     |     |   1    | 0-2 |   0    | 0-9 |  60  |   1   |
///     |     |   1    | 0-2 |   0    | A-F |  66  |   1   |
///     |     |   1    | 0-3 |   1    | 0-3 |  66  |   1   |
///     ----------------------------------------------------
///     | SUB |   0    | 0-9 |   0    | 0-9 |  00  |   0   |
///     ----------------------------------------------------
///     | SBC |   0    | 0-8 |   1    | 6-F |  FA  |   0   |
///     ----------------------------------------------------
///     | DEC |   1    | 7-F |   0    | 0-9 |  A0  |   1   |
///     ----------------------------------------------------
///     | NEG |   1    | 6-7 |   1    | 6-F |  9A  |   1   |
///     ====================================================
///     S is set if most-significant bit of the A is 1 after an
///     operation; otherwise, it is reset.
///     Z is set if A is 0 after an operation; otherwise, it is reset.
///     H: see the DAA instruction table.
///     P/V is set if A is at even parity after an operation;
///     otherwise, it is reset.
///     N is not affected.
///     C: see the DAA instruction table.
///     =================================
///     | 0 | 0 | 1 | 0 | 0 | 1 | 1 | 1 | 0x27
///     =================================
///     T-States: 4 (4)
///     Contention breakdown: pc:4
/// </remarks>
private void Daa()
{
    var daaIndex = _registers.A + (((_registers.F & 3)
        + ((_registers.F >> 2) & 4)) << 8);
            _registers.AF = s_DaaResults[daaIndex];
}
```

As the table embedded into the comment suggests, the tough job is to create the `s_DaaResults` helper table. The current implementation of the calculation method is about hundred lines of code.

Many other instructions are worth to mention. In the next post, you will learn implementation details about interrupt handling, I/O, and block transfer instructions.