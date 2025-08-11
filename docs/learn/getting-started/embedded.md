---
sidebar_position: 6
draft: false
---

# Embedded

> Introduction to Embedded Systems Programming with Ada

## Foreword

Embedded systems are software systems that run on devices with limited resources.
These systems often have specific requirements for reliability, real-time
operation, limited power consumption, and so on.

The Ada language is an excellent choice for implementing such systems due to
its capabilities. As a compiled language, Ada programs are very energy-efficient.
The language's original focus on reliability and safety aligns well with the goals
of embedded systems development.
Flexible representation specification capabilities allow for easy low-level
hardware interaction.
At the same time, the powerful type system enables natural expression of
application domain concepts.
Finally, the language standard includes a dedicated section for supporting
embedded and real-time systems.
By implementing the standard's requirements, Ada makes it possible to program
devices without the need for general-purpose operating systems, while offering
effective support for a multitasking environment on devices with quite limited
capabilities.

The purpose of this article is to help beginners understand the available tools
for programming embedded systems in Ada, using a simple program as an example.
We will cover the installation and configuration of the software, using the
`STM32F407` development board as a case study.

## Required Tools and Components

For embedded systems development, we need the following tools:

- Cross-compiler
- Runtime Library (RTL)
- Driver libraries (e.g., **Ada Drivers Library**)

### Cross-compiler

A cross-compiler allows you to build executable modules for a processor that
is different from the current one.
For example, building code for ARM using a workstation with an AMD64 architecture.
The Alire package manager has several ready-to-use cross-compilers for the
following processor architectures:

- ARM
- AVR
- RISC-V
- xtensa_esp32 (experimental, not recommended for beginners)

### Runtime Library (RTL)

A runtime library is required to support non-trivial language constructs.
When the compiler finds such a construct in the user's code, it looks for the
corresponding types and subprograms in the runtime library and uses them.
Logically, the RTL is closely related to the cross-compiler, and a new version
of the compiler requires a corresponding version of the runtime library.

Not all Ada language constructs are in demand in embedded systems.
For example, exception handling is considered too expensive to be fully
implemented.
Depending on the language constructs implemented, runtime libraries are divided
into several types.
Currently, three types of RTL are available with GNAT:

- `light` - a minimal version of the library, without support for multitasking
  and other constructs;
- `light-tasking` - with limited support for multitasking, the so-called
  Ravenscar, Jorvik profiles;
- `embedded` - a more complete runtime library.

Runtime libraries are designed for use with specific hardware, as they contain
initialization code and resource allocation.
Cross-compilers for ARM and RISC-V come with a set of RTLs for the most popular
development devices.
Support for other devices can be distributed as separate Alire crates.

### Driver Libraries

Embedded system microprocessors tend to have a rich assortment of peripheral
devices, such as general-purpose input/output ports, timers, UART, SPI, I2C,
etc.
Corresponding drivers are needed to work with these devices.
The **Ada Drivers Library** (ADL) project probably boasts the most complete
driver support.
ADL is distributed under a BSD license and contains driver code for microprocessor
devices, as well as drivers for external devices such as sensors, displays,
cameras, etc.
The ADL repository also provides examples of using drivers for some development
boards.

Unfortunately, ADL does not support Alire, and vice versa.
Part of the ADL code has been copied into Alire as separate crates.

## Practical Example: `STM32F407`

Let's consider creating a minimal application using the `STM32F407`
microcontroller as an example.
There are several development boards on the market based on this inexpensive
processor, from the `Discovery Board` to the Chinese `STM32 F4VE`.
The runtime library for this chip is called `light-tasking-stm32f4`.

We will start by creating an application in the terminal without using an IDE
to show how everything works, and then we will focus separately on the support
for such projects in GNAT Studio.

### Creating a Project

Alire helps us create a suitable project template:

```shell
alr init --bin test
cd test
```

The next step is to specify the cross-compiler in the project dependencies:

```shell
alr with gnat_arm_elf
```

For the project manager gprbuild to know that a cross-compiler should be used,
it is necessary to specify the `Target`.
The easiest way to do this is in the project file. At the same time, we will
specify which runtime library we need. Let's edit `test.gpr` by adding the
following lines:

```
for Target use "arm-eabi";
for Runtime ("Ada") use "light-tasking-stm32f4";
```

We will make the program code (`test.adb`) as simple as possible:

```ada
with Ada.Text_IO;

procedure Test is
begin
   loop
      Ada.Text_IO.Put_Line ("Hello");
      delay 1.0;
   end loop;
end Test;
```

Everything is ready to build the executable file:

```shell
alr build
```

Let's convert the resulting ELF into a binary file suitable for uploading
the program:

```shell
alr exec -- arm-eabi-objcopy -Obinary bin/test bin/test.bin
```

For uploading programs to devices based on STM32 processors, the **ST Link v2**
hardware debugger is usually used.
The Discovery board has it integrated directly on the board.
(**DAPLINK** will also work).
To control the hardware debugger, you can use the `st-util` software,
or the open-source package **OpenOCD** (which can also control **DAPLINK**).  
The command to upload the program with `st-util`:

```shell
st-flash --connect-under-reset write bin/test.bin 0x08000000
```

or with **OpenOCD**

```shell
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg -c 'program bin/test.bin verify reset exit 0x08000000'
```

Let's use the debugger to see how our program is executed.
In one console window, activate the hardware debugger:

```shell
st-util --semihosting
```

or (in the case of OpenOCD):

```shell
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
```

And in another, launch GDB:

```shell
alr exec arm-eabi-gdb ./bin/test
```

```gdb
target extended-remote :4242
```

or (for **OpenOCD**)

```gdb
target extended-remote :3333
monitor arm semihosting enable
```

In the debugger console, we can:

- execute the program step-by-step using the commands `next`, `step`;
- view the contents of registers `info reg`;
- check the memory contents `x <addr>`;
- perform a re-initialization of the processor `monitor reset init`.

## Integration with GNAT Studio

To configure GNAT Studio to launch `st-util` for program upload and debugging,
it is enough to add the following code to the project file:

```
package Ide is
   for Program_Host use "localhost:3333";
   for Communication_Protocol use "remote";
   for Connection_Tool use "st-util";
end Ide;
```

Now let's start the studio:

```shell
alr exec gnatstudio
```

The studio will recognize that this project is for embedded systems and will
allow you to build, upload the program, and debug it using the GUI.
Using OpenOCD with GNAT Studio is only slightly more difficult - you need to
prepare a configuration file, for example `openocd.cfg`:

```
source [find interface/stlink.cfg]
source [find target/stm32f4x.cfg]
reset_config none
```

Then, you need to reference this file in the IDE package of the project file:

```
package Ide is
   for Program_Host use "localhost:4242";
   for Communication_Protocol use "remote";
   for Connection_Tool use "openocd";
   for Connection_Config_File use "openocd.cfg";
end Ide;
```

## LED Example

Among the demo projects in GNAT Studio, there is one that might be of interest
to us. Let's create it right now.
Click /File/New Project and select "STM32F4 compatible/LED demo project".
When the studio creates the file, correct the name of the runtime library:

```
for Runtime ("Ada") use "light-tasking-stm32f4";
```

This example is designed for OpenOCD, but it is easy to reconfigure it to use
`st-util`, just correct the `Connection_Tool` in the `IDE` package in the
project file.  
This demo project contains code for controlling I/O ports.
It would be quite informative to understand the structure of this code, but
it is beyond the scope of an introductory article.
For now, we will only note that this code is not included in the runtime
library.
Drivers similar to this code were collected in a separate project, the
Ada Drivers Library (ADL).

## Ada Drivers Library

The Ada Drivers Library project was created quite a long time ago.
It contains drivers for various devices, from devices located on microcontrollers
to drivers for external devices such as sensors, displays, and even cameras.
Let's download it from GitHub:

```shell
git clone https://github.com/AdaCore/Ada_Drivers_Library
```

It is recommended to start getting acquainted with the examples provided in
this project.
Choose the appropriate device in the examples directory.
For example:

```
examples/stm32_f4ve/blinky_stm32_f4ve.gpr
```

Unfortunately, ADL does not support Alire, and vice versa.
Therefore, before opening the project in GNAT Studio, you need to make sure
that the cross-compiler and `gprbuild` are present in the `PATH`.
The easiest way is to run alr exec bash or alr exec gnatstudio in the test
directory we created at the beginning of the article.
This way, Alire will add the necessary paths to the `PATH` environment variable.
After that, you can open the example project in the studio.

### Connecting via `alr --pin`

Starting with Alire 2.1.0, you can connect project files in repository subdirectories
as crates, even if the `alire.toml` files are missing.
We can use this to connect the Ada Drivers Library.
To do this, we need to know the name of the project file and the directory
where it is located in the Ada Drivers Library (see Appendix 2).
For example, `stm32_f4ve_sfp.gpr` is located in the `boards/stm32_f4ve` directory.
Therefore, we can connect it like this:

```shell
alr pin stm32_f4ve_sfp
  --use=https://github.com/AdaCore/Ada_Drivers_Library
  --subdir=boards/stm32_f4ve
```

Now the drivers and board description are available to us.
So we can run the LED blinking:

```ada
with STM32.Board;
with STM32.GPIO;

procedure Test is
begin
   STM32.Board.Initialize_LEDs;

   loop
      STM32.GPIO.Toggle (STM32.Board.All_LEDs);

      delay 0.5;
   end loop;
end Test;
```

## Conclusion

In this article, we covered the basics of embedded systems programming in Ada,
using the `STM32F407` board as an example.
We got acquainted with the necessary tools, such as the cross-compiler,
runtime library, and driver libraries, and learned how to use them to create
simple programs.

We also looked at the process of building, uploading, and debugging Ada
programs, both using the command line and with the integrated development
environment GNAT Studio.
In addition, we learned how to use the Ada Drivers Library to work with
the microcontroller's peripheral devices.

We hope that this article will help beginner developers take their first
steps into the world of embedded systems programming in Ada.
Ada is a powerful and reliable language that is ideal for developing
critically important applications, and we encourage you to explore its
capabilities further.

## Appendix 1. List of supported ARM devices

### Embedded and/or light-tasking:

| Name              |
| :---------------- |
| feather_stm32f405 |
| microbit          |
| nrf52832          |
| nrf52833          |
| nrf52840          |
| nucleo_f401re     |
| openmv2           |
| rpi2              |
| rpi-pico          |
| rpi-pico-smp      |
| sam4s             |
| samg55            |
| samv71            |
| stm32f4           |
| stm32f429disco    |
| stm32f469disco    |
| stm32f746disco    |
| stm32f769disco    |
| tms570            |
| tms570lc          |
| zynq7000          |

### Only light RTL:

| Name         |
| :----------- |
| cortex-m0    |
| cortex-m0p   |
| cortex-m1    |
| cortex-m23   |
| cortex-m3    |
| cortex-m33df |
| cortex-m33f  |
| cortex-m4    |
| cortex-m4f   |
| cortex-m7df  |
| cortex-m7f   |
| lm3s         |

## Appendix 2. List of supported Ada Drivers Library boards

| Board (directory)   | light-tasking               | embedded                     | light                 |
| :------------------ | :-------------------------- | :--------------------------- | :-------------------- |
| crazyflie           | crazyflie_sfp.gpr           | crazyflie_full.gpr           |                       |
| feather_stm32f405   | feather_stm32f405_sfp.gpr   | feather_stm32f405_full.gpr   |                       |
| HiFive1             |                             |                              | hifive1_zfp.gpr       |
| HiFive1_rev_B       |                             |                              | hifive1_rev_b_zfp.gpr |
| MicroBit            |                             |                              | microbit_zfp.gpr      |
| NRF52_DK            |                             |                              | nrf52_dk_zfp.gpr      |
| nucleo_f446ze       | nucleo_f446ze_sfp.gpr       | nucleo_f446ze_full.gpr       |                       |
| OpenMV2             | openmv2_sfp.gpr             | openmv2_full.gpr             |                       |
| stm32f407_discovery | stm32f407_discovery_sfp.gpr | stm32f407_discovery_full.gpr |                       |
| stm32f429_discovery | stm32f429_discovery_sfp.gpr | stm32f429_discovery_full.gpr |                       |
| stm32f469_discovery | stm32f469_discovery_sfp.gpr | stm32f469_discovery_full.gpr |                       |
| stm32_f4ve          | stm32_f4ve_sfp.gpr          | stm32_f4ve_full.gpr          |                       |
| stm32f4xx_m         | stm32f4xx_m_sfp.gpr         | stm32f4xx_m_full.gpr         |                       |
| stm32f746_discovery | stm32f746_discovery_sfp.gpr | stm32f746_discovery_full.gpr |                       |
| stm32f769_discovery | stm32f769_discovery_sfp.gpr | stm32f769_discovery_full.gpr |                       |
| stm32_h405          | stm32_h405_sfp.gpr          | stm32_h405_full.gpr          |                       |
| Unleashed           | unleashed_sfp.gpr           | unleashed_full.gpr           | unleashed_zfp.gpr     |
