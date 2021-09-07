# Writing Renode C# Peripherals

Renode is written in C#, which means it has access to the entire base of C#. One feature of C# is the `CSharpCodeProvider` object which provides the `CompileAssemblyFromSource(CompilerParameters, string[])` function. This means that Renode has a runtime C# compiler built in.

You can `include` C# files in the Renode console or in your startup script to dynamically add new peripherals to your environment. Xous uses this extensively in Betrusted since the hardware peripherals are still under development and therefore change regularly. Updating a hardware module in Renode simply involves modifying the `.cs` file and restarting Renode. There is no additional compile step.

## Setting up an IDE -- Visual Studio Code

It is highly recommended to use a full IDE. The Renode API can change, and it can take time to restart Renode to recompile your C# files. An IDE will provide you with tab-completion and will immediately tell you if there is a code error.

The core of Renode is written in a full IDE such as Visual Studio or Monodevelop. These IDEs expect a full Project file that defines a single target output -- for example an executable or a linked library. With our usage of C# there is no single target since Renode will dynamically load the source files. To work around this, we create a stub project file that tricks the IDE into loading our assembly files and providing autocomplete. We never actually *use* this project file, but it's used behind the scenes automatically.

Broadly speaking, there are three steps to setting up an IDE:

1. Download Visual Studio Code
2. Copy the reference project file
3. Modify the reference project file
4. Install the C# extension.

To begin with download [Visual Studio Code](https://code.visualstudio.com/). It is available for Windows, Linux, and Mac.

Next, copy `emulation/peripherals.csproj.template` to `emulation/peripherals.csproj`. This is a C# Project file that is understood by Visual Studio and Visual Studio Code. The file name `peripherals.csproj` is in the `.gitignore` file, so don't worry about accidentally checking it in.

Edit `peripherals.csproj` and modify `RenodePath` to point to your Renode installation where the `.dll` files are located. On Linux this is likely `/opt/renode/bin`. On Windows this may be in `C:\Program Files\`.

Finally, install the [C# for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp) extension. This extension will activate, parse your `.csproj`, and start providing autocomplete and compile suggestions.

## Creating a new Peripheral

To create a new peripheral, simply copy an existing peripheral to a new filename under `emulation/peripherals/`, making sure the filename ends in `.cs`.

Many examples exist in the `emulation/peripherals/` directory, and you can find many more examples [built into Renode](https://github.com/renode/renode-infrastructure/tree/master/src/Emulator/Peripherals/Peripherals).

A simple example could be a device that provides random numbers:

```cs
using Antmicro.Renode.Core;
using Antmicro.Renode.Core.Structure.Registers;
using Antmicro.Renode.Logging;

namespace Antmicro.Renode.Peripherals.Miscellaneous
{
    public class ExampleRNGServer : BasicDoubleWordPeripheral, IKnownSize
    {
        public long Size { get { return 0x100; } }
        public GPIO IRQ { get; private set; }
        private readonly PseudorandomNumberGenerator rng = EmulationManager.Instance.CurrentEmulation.RandomGenerator;
        private bool enabled = true;

        private enum Registers
        {
            CONTROL = 0x0,
            DATA = 0x4,
            STATUS = 0x8,
            AV_CONFIG = 0xc,
            RO_CONFIG = 0x10,

            READY = 0xc4,
            EV_STATUS = 0xc8,
            EV_PENDING = 0xcc,
            EV_ENABLE = 0xd0,
            URANDOM = 0xdc,
            URANDOM_VALID = 0xe0,
            TEST = 0xf8,
        }

        public ExampleRNGServer(Machine machine) : base(machine)
        {
            this.IRQ = new GPIO();
            DefineRegisters();
        }


        private void DefineRegisters()
        {

            Registers.URANDOM.Define(this)
                .WithValueField(0, 32, FieldMode.Read, valueProviderCallback: _ =>
                {
                    if (!enabled)
                        return 0;
                    return (uint)rng.Next();
                }, name: "URANDOM");
            Registers.DATA.Define(this)
                .WithValueField(0, 16, FieldMode.Read, valueProviderCallback: _ =>
                {
                    if (!enabled)
                        return 0;
                    return (uint)rng.Next();
                }, name: "DATA")
                .WithValueField(16, 16, FieldMode.Read, valueProviderCallback: _ =>
                {
                    return 0xf00f;
                }, name: "SIGNATURE");
            Registers.URANDOM_VALID.Define(this)
                .WithFlag(0, FieldMode.Read, valueProviderCallback: _ => { return true; }, name: "URANDOM_VALID")
                .WithFlag(1, FieldMode.Read, valueProviderCallback: _ => { return enabled; }, name: "ENABLE");

            Registers.CONTROL.Define(this)
                .WithFlag(0, FieldMode.Write, writeCallback: (_, val) => { enabled = val; }, name: "ENABLE");
        }
    }
}
```

There's a lot to take in there, particularly if you've never dealt with C# before. Let's go over the module line-by-line.

```cs
using Antmicro.Renode.Core;
using Antmicro.Renode.Core.Structure.Registers;
using Antmicro.Renode.Logging;
```

The first three lines import various packages to the current namespace. You'll most likely use these in all of your projects. Any valid C# namespace may be used, including core `.Net` libraries. This can be useful if you need networking, cryptography, or other exotic libraries. There are many useful logging functions as well. You'll notice that the final line is darker than the other two. This is because this package is currently unused -- we don't perform any logging currently. You can safely remove this final line, however it's useful to leave Logging as an import because it allows for autocompletion of Logging functions.

```cs
namespace Antmicro.Renode.Peripherals.Miscellaneous {
```

Next, we define the namespace for this module. The module MUST be under a `namespace Antmicro.Renode.Peripherals.xxx` namespace. In this case, it is under `Antmicro.Renode.Peripherals.Miscellaneous`. This namespacing provides a handy structure to various peripherals.

```cs
public class ExampleRNGServer : BasicDoubleWordPeripheral, IKnownSize {
```

Finally, we begin to define our class. This class is named `ExampleRNGServer`, and it inherits `BasicDoubleWordPeripheral` and `IKnownSize`.

The `BasicDoubleWordPeripheral` class provides several convenience functions that makes it easy to create a memory-mapped device. It means we don't need to manage accessors, and we can simply worry about the register values themselves.

Peripherals need to have a known size, so we inform C# that our client has a known size. The `I` stands for Interface. To find out which functions we must implement to conform to `IKnownSize`, hold Ctrl and click on `IKnownSize`. It will take you to the definition of `IKnownSize`, located inside `Emulator.dll`. You will note that the only thing we need to implement is `log Size { get; }`, which means we only need to create an accessor for the property `Size`.

```cs
public long Size { get { return 0x100; } }
public GPIO IRQ { get; private set; }
private readonly PseudorandomNumberGenerator rng = EmulationManager.Instance.CurrentEmulation.RandomGenerator;
private bool enabled = true;
```

Here we define our local properties and variables. We can see the `Size` property defined here. Our peripheral goes up to `0xf8`, so we return that as a constant. This is used by Renode to ensure peripherals don't overlap, and to know which peripheral to invoke when memory is accessed.

There is an IRQ here as well, which is a `GPIO`. The way Renode handles interrupts is by reusing GPIO pins. We can trigger an interrupt by setting this GPIO, and the system will invoke an interrupt context on the CPU.

Finally there is a local variable that is part of this object and not visible outside of our class.

```cs
private enum Registers
{
    CONTROL = 0x0,
    DATA = 0x4,
    STATUS = 0x8,
    AV_CONFIG = 0xc,
    RO_CONFIG = 0x10,

    READY = 0xc4,
    EV_STATUS = 0xc8,
    EV_PENDING = 0xcc,
    EV_ENABLE = 0xd0,
    URANDOM = 0xdc,
    URANDOM_VALID = 0xe0,
    TEST = 0xf8,
}
```

We define an enum called `Registers`. This is simply a mapping of register names to register numbers. It is not a particularly special enum, however correct naming of the enum values will make it easier to define the register set later on. It is standard practice to define all possible registers in this enum, even if you do not implement them right away.

```cs
public ExampleRNGServer(Machine machine) : base(machine)
{
    this.IRQ = new GPIO();
    DefineRegisters();
}
```

This is the constructor for our device. It takes a single argument of type `Machine`. Because we inherit from `BasicDoubleWordPeripheral`, we will need to call the constructor for the base class. To figure out what the constructor looks like, hold Ctrl and click on `BasicDoubleWordPeripheral`. We can see that the constructor for that class simply takes one argument that's a `Machine`. Therefore, the first line of our constructor should invoke the base constructor directly. Which is what we do here.

We create a new GPIO and assign it to the IRQ. Renode will access our `IRQ` property if it wants to watch for interrupts. If our peripheral has no interrupts we can omit the `IRQ` property.



```cs
private void DefineRegisters() {
```
Finally, we invoke the `DefineRegisters()` function. It is the most complicated function in this class, however it's where most of the work is done. Let's look at each register definition in order.

```cs
Registers.URANDOM.Define(this)
    .WithValueField(0, 32, FieldMode.Read, valueProviderCallback: _ =>
    {
        if (!enabled)
            return 0;
        return (uint)rng.Next();
    }, name: "URANDOM");
```
The `Define(this)` function comes from `BasicDoubleWordPeripheralExtensions`, which is one of the classes provided to us as a subclass of `BasicDoubleWordPeripheral`. It allows us to define a register on an enum type.

The `WithValueField()` function defines a value for a register across a range of values. In this case, we define a value beginning at bit 0 that is 32-bits wide. We define this register as a `FieldMode.Read` register, meaning writes will be ignored. When a device accesses this register, the `valueProviderCallback` function will be called.

What follows is a C# closure. The first argument is the register itself, which we ignore since we are not interested in it. Therefore, the variable is named `_`. If the block is disabled, we return 0, otherwise we return a `uint` from the class RNG provider.

Finally, we name the register `URANDOM`.

```cs
Registers.DATA.Define(this)
    .WithValueField(0, 16, FieldMode.Read, valueProviderCallback: _ =>
    {
        if (!enabled)
            return 0;
        return (uint)rng.Next();
    }, name: "DATA")
    .WithValueField(16, 16, FieldMode.Read, valueProviderCallback: _ =>
    {
        return 0xf00f;
    }, name: "SIGNATURE");
```

This register contains two value fields. The first is at offset 0, and is 16-bits wide. The second is at offset 16, and is also 16-bits wide.

The `valueProviderCallback` function is called for each field, which avoids the need for any manual bit shifting.

If the peripheral is not enabled, then the `DATA` field returns 0. If it is enabled, then it returns a 16-bit random value.

Because of the way this register is defined, the top 16 bits will always be `0xf00f`. Therefore, the register's value will be either `0xf00f0000` or `0xf00fRAND`.

```cs
Registers.URANDOM_VALID.Define(this)
    .WithFlag(0, FieldMode.Read, valueProviderCallback: _ => { return true; }, name: "URANDOM_VALID")
    .WithFlag(1, FieldMode.Read, valueProviderCallback: _ => { return enabled; }, name: "ENABLE");
```

This register defines two flags. The first flag is at bit 0, and the second flag is at bit 1. Flags are always one-bit boolean values, which is why the `valueProviderCallback` returns `true` instead of a `uint` like we've seen in the past. Similarly to `WithValueField()`, a `WithFlag` value will call the `valueProviderCallback` for each flag, avoiding the need to do complex shifting.

```cs
Registers.CONTROL.Define(this)
    .WithFlag(4, FieldMode.Write, writeCallback: (_, val) => { enabled = val; }, name: "ENABLE");
```

Finally we define the `CONTROL` register. Our implementation simply has an `ENABLE` bit at offset 4. This is the first time we've seen a `writeCallback`. This closure takes two arguments: the register itself and the written value. 

## Using the new peripheral

To use the new peripheral, save it in a `.cs` file, then include it in Renode. For example, if it was called `examplerngserver.cs`, you would include it in Renode by running:

```text
(renode) i @examplerngserver.cs
```

You can then use the `Miscellaneous.ExampleRNGServer` peripheral in any platform definition. For example, to create a new peripheral at offset `0x40048000` in the current machine, use the `LoadPlatformDescriptionFromString` command:

```text
(renode) machine LoadPlatformDescriptionFromString 'rng: Miscellaneous.ExampleRNGServer @ sysbus 0x40048000'
```

Now, any accesses to `0x40048000` will be directed to your new peripheral.