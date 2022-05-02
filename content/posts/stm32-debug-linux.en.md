+++
title = "STM32 Debugging using OpenOCD, GDB & GDBFrontend on Linux"
date = 2022-03-19T13:04:02-06:00
lastmod = 2022-03-19T13:04:02-06:00
tags = ["STM32", "Linux", "GDB", "OpenOCD"]
categories = ["Embedded"]
imgs = []
cover = "/img/covers/stm32_dev_debug.png"
readingTime = true
toc = true
comments = false
justify = false
single = false
license = ""
draft = false
[twitter]
  card = "summary_large_image"
  site = "@jhestolano"
  title = "STM32 Debugging using OpenOCD, GDB & GDBFrontend on Linux"
  description = "In this post I will show you how you could use GDB & OpenOCD to debug an STM32 microcontroller on Linux"
  image = "https://elrobotista.com/img/covers/stm32_dev_debug.png"
+++

In previous posts I wrote about how you could easily start an embedded project for an STM32 micro using open source tools on Linux. In this one, I will go onto the next step: How could we do actual debugging using open source tools? I am not talking about simple ```printf``` statements; although useful, this debugging technique is somewhat limited when you need to look into detail on what is going on at register configuration level. It poses some challenges as you may not have an UART port available. Maybe the issue happens way before the UART is even setup (startup code). Maybe the bug doesn't even show up when using ```printf``` statements because the timing of the code execution is affected.

For these reasons, I want to show you how you could setup a debugging environment using [GDB](https://www.sourceware.org/gdb/), [Opencd](https://openocd.org/) y [GDBFrontend](https://github.com/rohanrhu/gdb-frontend).

## OpenOCD

"Open On-Chip Debugger" is a debugging, programming & boundary scanning tool for multiple platforms, such as:

* ARMv5 through latest ARMv8
* MIPS
* AVR (incomplete)
* Andes
* RISC-V

It's role in our plans is to use it in tandem with GDB to debug an ARM Cortex-M4. To install it, launch a terminal and type:
``` none
git clone https://github.com/openocd-org/openocd
cd openocd
./bootstrap
./configure
make && sudo make install
```
Done. To verify it was correctly installed, try the ```openocd --version``` command. The output should be similar to this:

``` none
❯ openocd --version
Open On-Chip Debugger 0.11.0+dev-00615-gbe0d68eb6 (2022-03-12-20:07)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
```

Using OpenOCD for our application is fairly simple, we only need to provide two arguments: the configuration file for the debugger (st-link) & the microcontroller description file. In this example, I'm using a [NUCLEO-F303RE](https://www.st.com/en/evaluation-tools/nucleo-f303re.html), with an [STLINK](https://www.st.com/en/development-tools/st-link-v2.html). Thus, the command looks as follows:
``` none
openocd -f /usr/local/share/openocd/scripts/interface/stlink.cfg -f /usr/local/share/openocd/scripts/board/st_nucleo_f3.cfg
```

Feel free to explore your ```/usr/local/share/openocd``` folder to look at what other description / configuration files you can find.

One nice feature of NUCLEO boards is the integrated STLINK debugger:

![Embedded STLINK](/img/posts/nucleo_stlink.jpg#center)

Launching OpenOCD, the server is started and the following message will be printed to the console:
```none
Open On-Chip Debugger 0.11.0+dev-00615-gbe0d68eb6 (2022-03-12-20:07)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Warn : Interface already configured, ignoring
Error: already specified hl_layout stlink
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
Info : DEPRECATED target event trace-config; use TPIU events {pre,post}-{enable,disable}
srst_only separate srst_nogate srst_open_drain connect_deassert_srst

Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : clock speed 1000 kHz
Info : STLINK V2J30M19 (API v2) VID:PID 0483:374B
Info : Target voltage: 3.244649
Info : [stm32f3x.cpu] Cortex-M4 r0p1 processor detected
Info : [stm32f3x.cpu] target has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32f3x.cpu on 3333
Info : Listening on port 3333 for gdb connections
```
You should even see the ST-LINK LED (LD1) blinking, alternating between red and green colors.

![OpenOCD STM32](/img/posts/stm32_openocd_blink.gif#center)

Done! Notice the GDB server is waiting for a connection to ```localhost :3333```.

## GDB

Installing GDB using your distro's package manager should be the simplest way to get it done. The problem here is that GDBFrontend expects GDB compiled with Python3 support and other specific options. At least in my distro (Artix) I could not find a GDB option with everything I needed (if you do, feel free to use that one, and skip this section). So let's install it from source. It is not hard, but it takes some time. For my setup it was about 15 minutes.

To compile GDB, the steps are well defined here: https://github.com/rohanrhu/gdb-frontend/wiki/Embedded-Debugging. I will put them next for your convenience:
``` none
# Download source.
wget https://ftp.gnu.org/gnu/gdb/gdb-11.2.tar.xz

# Unzip source folder.
tar zxvf gdb-11.2.tar.gz

# Create build folder to launch process from there. Notice the compile flags.
mkdir gdb-11.2-build
cd gdb-11.2-build
../configure --with-python=/usr/bin/python3 --target=arm-none-eabi --enable-interwork --enable-multilib

# Let's go!
make
```

Note the configuration flags reference ```/usr/bin/python3```, so we are assuming you already have installed the correct Python version, at the default path. If for some reason, your Python is installed elsewhere, make sure to update the installation path. If you are not sure where it is installed, or you just want to make sure, run:
```none
❯ which python3
/usr/bin/python3
```

Once the compilation process is done, you can always check if it looks correct using the GDB version command:```cd gdb/ && ./gdb --version```:
```none
GNU gdb (GDB) 11.2
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

This is just a quick some test. To make sure it actually worked, let's try to connect OpenOCD and GDB. Assuming OpenOCD is already running (using the command mentioned above), run the following command: ```./gdb``` from the ```gdb-11.2-build/gdb``` folder. We are now inside GDB. To connect to the server, run ```target extended-remote:3333```. Note we are referencing the port shown by OpenOCD (if this your OpenOCD uses a different port, please update it). You should see something similar to this:
``` none
❯ ./gdb
Python Exception <class 'ModuleNotFoundError'>: No module named 'gdb'
./gdb: warning:
Could not load the Python gdb module from `/usr/local/share/gdb/python'.
Limited Python support is available from the _gdb module.
Suggest passing --data-directory=/path/to/gdb/data-directory.
Exception caught while booting Guile.
Error in function "open-file":
No such file or directory: "/usr/local/share/gdb/guile/gdb/boot.scm"
./gdb: warning: Could not complete Guile gdb module initialization from:
/usr/local/share/gdb/guile/gdb/boot.scm.
Limited Guile support is available.
Suggest passing --data-directory=/path/to/gdb/data-directory.
GNU gdb (GDB) 11.2
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pc-linux-gnu --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb) target extended-remote :3333
Remote debugging using :3333
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
Python Exception <class 'NameError'>: Installation error: gdb._execute_unwinders function is missing
0x080001be in ?? ()
(gdb)
```
We are not completely through yet, there are some issues with our configuration. Let's put them aside for a moment; we will get there in the next section.

## GDBFrontend

So the motivation to play around with GDBFrontend is due to the fact that gdb, although a very powerful debugger, the learning curve can be a struggle if you are not familiar with terminal applications (which is almost always the case when you are a junior developer). For this reasons there are tools such as [GDB dashboard](https://github.com/cyrus-and/gdb-dashboard), that try to soften the learning curve making the interface a little bit easier and more human friendly to use with Python:

![GDB Dashboard](/img/posts/gdb_dash.jpg#center)

The problem here is that, even though the interface now looks nicer, the interface to GDB still remains a 100% console application. A graphical frontend like GDBFrontend (Web browser based) softens the learning curve and allows us to quickly get familiar with how the tool actually works. Let's take a look at the installation process.

Clone the project: ```git clone https://github.com/rohanrhu/gdb-frontend```. This tool is Python based, so there is not compiling to do (another nice feature, right?) cd into the tool folder and run ```./gdbfrontend```. GDBFrontend should automatically open a new tab in your web browser, and it should be ready to use! By default, the address to connect to it is ```http://127.0.0.1:5550/terminal/```. You should see a blue window with the legend "No opened source":

![Default GDBFrontend](/img/posts/gdb-frontend-default.jpg#center)

We are ready to start debugging some code!

## Debugging

Let's go back to the 'blinky' project we started in a [previos post](/posts/stm32-dev-linux). You can clone the project and build it like so (assumes you have [stlink tools](https://github.com/stlink-org/stlink) installed):

``` none
git clone https://github.com/elrobotista/stm32_dev_linux && cd stm32_dev_linux
git submodule update --init
cd libopencm3 && make
cd ../appl & make
st-flash --reset write blink.bin 0x8000000
```
Feel free, of course, to substitute this code with your own code if you have already something setup. Just make sure to have the debugging symbols handy (.elf file).

If you're using my 'blinky' project, you should see a green LED flashing approximately every second. Let's begin debugging it using GDBFrontend. Once more, start OpenOCD if you haven't already done so (remember tu use the correct .cfg files for your processor):

``` none
openocd -f /usr/local/share/openocd/scripts/interface/stlink.cfg -f /usr/local/share/openocd/scripts/board/st_nucleo_f3.cfg
```

Next step: launch GDBFrontend. Remember the issue / error we previously had above? This can be fixed easily following the steps mentioned in the GDBFrontend repository. We need to specify the 'data directory'. Running GDBFrontend like so:

```none
~/Repos/gdb-frontend/gdbfrontend -g $(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/gdb) -G --data-directory=$(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/data-directory/)
```

should fix the issue.

This command references ```gdbfrontend``` executable, which lives in folder ```Repos``` (feel free to use whatever location works for you on your file system). In a similar fashion, ```gdb``` was compiled in folder ```Repos/gdb-11.2/```. This is precisely what was missing: ```gdbfrontend``` needed to know where ```gdb``` with ```python3``` and the ```data-directory``` actually lived in the file system. This is a pretty long command, so to avoid typing this over and over again, we can create a ```scripts``` folder in ```home``` and create a ```launch_gdbfrontend.sh``` script:
```none
mkdir ~/scripts
echo ~/Repos/gdb-frontend/gdbfrontend -g $(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/gdb) -G --data-directory=$(realpath ~/Repos/gdb-11.2/gdb-11.2-build/gdb/data-directory/) >> launch_gdbfrontend.sh
sudo chmod +x
```

Then, launching ```gdbfrontend``` is as simple as running ```~/scripts/launch_gdbfrontend.sh```. If OpenOCD is already running, you can click `connect` using `localhost:3333`:

![GDBFrontend Connect](/img/posts/gdbf-rename.jpg#center)

Next, load the executable with debugging symbols ```blinky.elf```:

![GDBFrontend Executable](/img/posts/gdbf-load-exec.jpg#center)

If you're correctly following these steps, you should already see the assembly instructions in the source window. Now you only need to navigate to the `source code` location so that GDBFrontend is able to match the source code to the instructions being executed. Let's also insert a breakpoint:

![GDBFrontend Breakpoint](/img/posts/gdbf-bkpt.jpg#center)

On the left side pane, you can see the file tree, where I navigated to ```~/Repos/stm32_dev_linux/appl/main.c```. And you are all set! Now you can debug your code in a friendly and visually easy to use manner.

Click on the ```step in``` arrow to go inside the ```gpio_toggle``` function. From there you can execute the code instruction by instruction to find the exact place where the LED is actually turned on (or off). Now imagine all you could do using this tool in your future projects! No more ```printf``` statements everywhere in your code!

On the right hand side pane you can see your local variables, its values, the function call stack (which basically tells you where your code is coming from) and processor registers:

![GDBFrontend Registers](/img/posts/gdbf-registers.png#center)

On the bottom pane, you have an actual GDB console available to run any command you would run on a normal GDB console. For example, let's say you are interested to know what's inside address ```0x20000000``` and the next 32 words (a word is 4 bytes). You would type ```x/32xw 0x20000000``` (which means examine the next 32 words in hex format starting from 0x20000000):

![GDBFrontend Dump](/img/posts/gdbf-dump.png#center)

If you're already familiar with GDB, and you already know your ```step, continue, next, nexti, break```, etc, commands, you could directly type them in the console as well the control execution flow.

Another interesting feature of GDBFrontend is the expression evaluator. You could perform some mathematical operations using local or global variables in context. Or maybe you can see some mathematical / logical operations in your code, and you're not completely sure what it evaluates to, you could try the expression evaluator. For example, line 90 in ```gpio_common_all.c``` contains the following line of code:
```c
  uint32_t port = GPIO_ODR(gpioport);
  GPIO_BSRR(gpioport) = ((port & gpios) << 16) | (~port & gpios);
```
The variable ```port``` has been optimized out. But we can still get the value using the evaluator:

![GDBFrontend Expression-1](/img/posts/gdbf-expression-1.png#center)

The next one, we could evaluate manually substituting the previous ```port``` value into the expression:

![GDBFrontend Expression-2](/img/posts/gdbf-expression-2.png#center)

What do you think? I believe this is a great tool and the developer(s) that created this are without a doubt very talented. I'm pretty sure this tool has some more nice features, but I'm still exploring it. This is all for this introduction and I hope you liked it. 'til the next one!
