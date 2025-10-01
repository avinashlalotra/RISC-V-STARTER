<ol>

<h1><li> Programming a Minimal SoC </h1>

In this chapter, we will revise the minimal SoC we had designed. Our SoC included a <b>systolic array</b> to accelerate matrix multiplication workloads. This SoC was implemented on an <b>Arty 7 FPGA board</b> . In this book, we will see how to program such a system.  

To program any embedded system, we need a few fundamental things that are common across all systems.  

<h2> Booting Example: Laptop, Computer, or Phone </h2>

Let us take an example of what happens when we turn on our laptop, computer, or phone.  

Our device has <b>firmware</b> hardcoded by the manufacturer. This firmware is stored in <b>ROM</b>, so it cannot be changed. The firmware makes all the hardware units ready and loads a program called the <b>bootloader</b>.  

<ul>
    <li> The bootloader then loads the <b>operating system (OS)</b>, if any, and configures various hardware components so they are available to the user.  </li>
    <li> The bootloader is stored in some <b>non-volatile memory</b>, such as EEPROM, SSD, or a disk.  </li>
</ul>

After the bootloader, if we have an OS, it transfers control to the OS. Otherwise, in bare-metal systems where there is no OS, it loads the <b> main program</b>, which then controls the execution.  


</li>
<!-- ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- -->
<h1><li>Our Minimal SoC </h1>

Now let's return to our SoC. It consists of:  

<ul>
    <li><b>Picorv32 core</b> </li>  
    <li><b>RAM</b> </li>  
    <li><b>UART</b> </li>  
    <li><b>LEDs</b> </li>
    <li><b>Matrix Multiplication Unit</b> </li>    
  
</ul>


<img src="images/image.jpg" alt="Soc Diagram" />


To program an embedded system /SoC , you only need its <b> memory map </b>. This is because I/O peripherals are modeled as memory.  

So we can choose the memory map if we are the designers otherwise the vendor will provide you the details. In our case we are the designers so lets choose this memory map.

<img src="images/memoryMap.png" alt="Memory map"/>

And our RAM will have range from 0x0000 to 0x10000.


Since we will not use an  OS here ,so this is a bare-metal system. Therefore, we need three main components:  

<ul>
    <li> Firmware  </li>
    <li> Bootloader  </li>
    <li> Main program  </li>
</ul>

<h2>Understanding Each Component</h2>

<h3>Firmware </h3>

Firmware is the low-level software that initializes and tests the hardware when the system is powered on. It is usually stored in <b>ROM</b> or <b>Flash memory</b> and is responsible for making sure all hardware blocks (CPU, RAM, UART, peripherals, etc.) are ready to use.  

<ul>
    <li>In our SoC, the firmware will configure the Picorv32 core, initialize the RAM, set the initial states of LEDs, and prepare the matrix multiplication unit.  </li>
    <li> Firmware is also responsible for loading the bootloader from non-volatile memory into RAM so that it can be executed.  </li>
</ul>

<h3>Bootloader </h3>

The bootloader is a small program whose main function is to <b> load the main program </b>into memory and then transfer control to it.  

<ul>
    <li> It can perform basic hardware setup that the firmware hasn’t done, such as configuring UART for debugging or setting up memory-mapped peripherals.  </li>
    <li> In some systems, the bootloader can provide features like firmware updates or self-test routines before starting the main program.  </li>
</ul>

<h3> Main Program </h3>

The main program is the user-level application that actually performs the tasks intended for the system.  

<ul>
    <li> In our bare-metal SoC, the main program will control the execution flow, read inputs from UART or other peripherals, and perform matrix multiplication using our accelerator.</li>
    <li> It directly interacts with the memory-mapped peripherals using the memory addresses defined in the memory map.</li>

</ul>

<!-- ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- -->
<h1><li>Setting Up RISC-V Toolchain</h1>

So, what exactly is a <b>toolchain</b>?  

A toolchain is a collection of tools used for compiling, linking, and loading a program.  

Have you ever used a toolchain before?  
Think back to your Computer Programming Lab, where you used the <b>gcc</b> compiler for C programming.  
In fact, GCC itself is a toolchain—it includes a compiler (<b>gcc</b>), a debugger (<b>gdb</b>), and several other utilities.  

A toolchain is always architecture-specific. For example:  
- If you are using a laptop with an ARM processor (like Apple products), you need a version of the GCC toolchain that supports ARM.  
- If your laptop has an Intel processor, you need the GCC toolchain built for Intel’s architecture.  

Since we are working with a RISC-V processor, we need the RISC-V GCC toolchain.  

In this chapter, we will learn how to install the RISC-V GCC toolchain.  
For this, you need a Linux-based operating system such as Ubuntu. Any recent version of Ubuntu will work fine.  

If you are using Windows, you can first install a virtual machine (such as VirtualBox) and run Ubuntu inside it.  

You may also refer to the official GitHub repository for the RISC-V GNU Toolchain at: <a href="https://github.com/riscv-collab/riscv-gnu-toolchain"> GNU Toolchain</a>

if you prefer to set it up manually.  

However, to keep things simple, follow the instructions below.  

---

### Installation Steps (on Ubuntu)

1. Open a terminal and install the required dependencies:  

```bash
sudo apt update
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip python3-tomli \
libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool \
patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev libslirp-dev
```

2. Clone the RISC-V GNU Toolchain repository:  

```bash {copy = true}
git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
```



3. Configure and build the toolchain:  
```bash
./configure --prefix=/opt/riscv --with-arch=rv32imc --with-abi=ilp32d
make
```

This will install the RISC-V toolchain in your system under <b>/opt/riscv</b>.  

---

### Updating the PATH Variable

To make the RISC-V toolchain accessible system-wide, add it to your PATH:  

```bash
echo "export PATH=/opt/riscv/bin:$PATH" >> ~/.bashrc
source ~/.bashrc
```

---

### Verifying the Installation

Finally, check whether the toolchain is installed correctly:  

```bash
riscv64-unknown-elf-gcc --version
```

If you see version details printed, congratulations — the RISC-V GCC toolchain has been successfully installed!  

</li>


<h1> <li> Setting up RISC-V Simulator </h1> 

You might be wondering why we need a simulator at all.  

When you compile a C program on your system using the RISC-V toolchain, the output binary will not run on your computer.  
This is because it is compiled for the RISC-V architecture, not for your laptop’s Intel or ARM processor.  

So, do we need a real RISC-V processor to run it?  
Yes, ideally we would run it on hardware such as an FPGA board with a RISC-V core loaded on it.  

But what if we don’t have real RISC-V hardware available?  
In such a situation, we can use a <b>simulator</b> that mimics the behavior of a RISC-V processor and allows us to run our compiled program.  

Now you understand why we need a simulator here.  

There are a number of RISC-V simulators available, and in this chapter we will use <b>Spike</b>, the official RISC-V ISA simulator.  

---

### Installing Spike

Follow the steps below to install Spike on your system:  

```bash
sudo apt-get install device-tree-compiler libboost-regex-dev libboost-system-dev

git clone https://github.com/riscv-software-src/riscv-isa-sim
cd riscv-isa-sim
mkdir build
cd build
../configure --prefix=/opt/riscv
make
sudo make install
```

After installation, the <b>spike </b> binary will be available in </b> /opt/riscv/bin </b>.  

</ol>
