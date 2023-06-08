# seL4 Tutorial Docs

> 22.12.20

led by [Tutorials | seL4 docs](https://docs.sel4.systems/Tutorials/)

### Platform Development

Goal:

- You want to contribute to development of the seL4 (user-level) platform.
- You want to develop operating system services and device drivers.
- You want to develop seL4-based frameworks and operating systems.

Follow these tutorials:

1. [seL4 overview](https://sel4.systems/About/seL4-whitepaper.pdf)
2. [Introduction tutorial](https://docs.sel4.systems/Tutorials/#introduction-tutorial)
3. [seL4 mechanisms tutorial](https://docs.sel4.systems/Tutorials/#seL4-mechanisms-tutorials)
4. [Rapid prototyping tutorials](https://docs.sel4.systems/Tutorials/#rapid-prototyping-tutorials)
5. [CAmkES tutorials](https://docs.sel4.systems/Tutorials/#camkes-tutorials)
6. [Virtualisation tutorial](https://docs.sel4.systems/Tutorials/#virtual-machines)
7. [MCS tutorial](https://docs.sel4.systems/Tutorials/#mcs-extensions)



### Introduction tutorial

Before starting tutorials, make sure that you have read through the [Prerequisites](https://docs.sel4.systems/Tutorials/#prerequisites) and in particular [Doing the tutorials](https://docs.sel4.systems/Tutorials/#doing-the-tutorials).

Then, before any other tutorial, do the hello-world tutorial:

1. [Hello, World!](https://docs.sel4.systems/Tutorials/hello-world.html) an introduction to seL4 projects and tutorials.

which will ensure your setup is working correctly.

### seL4 mechanisms tutorials

This set of tutorials are for people keen to learn about the base mechanisms provided by the seL4 kernel. You will learn about the kernel API through small exercises that show basic examples.

1. [Capabilities](https://docs.sel4.systems/Tutorials/capabilities.html) an introduction to capabilities in the seL4 kernel API.
2. [Untyped](https://docs.sel4.systems/Tutorials/untyped.html) user-level memory management.
3. [Mapping](https://docs.sel4.systems/Tutorials/mapping.html) virtual memory in seL4.
4. [Threads](https://docs.sel4.systems/Tutorials/threads.html) how to start a thread using the seL4 API.
5. [IPC](https://docs.sel4.systems/Tutorials/ipc.html) overview of interprocess communication (IPC).
6. [Notifications](https://docs.sel4.systems/Tutorials/notifications.html) using notification objects and signals.
7. [Interrupts](https://docs.sel4.systems/Tutorials/interrupts.html) receiving and handling interrupts.
8. [Faults](https://docs.sel4.systems/Tutorials/fault-handlers.html) fault (e.g virtual memory fault) handling and fault endpoints.

### Rapid prototyping tutorials

This set of tutorials provides walkthroughs and exercises for using the dynamic libraries provided in [seL4_libs](https://docs.sel4.systems/Tutorials/TODO LINK). These libraries are provided as is, and have been developed for rapidly prototyping systems on seL4. They have not been through intensive quality assurance and do have bugs. Contributions are welcome.

1. [Dynamic Libraries 1](https://docs.sel4.systems/Tutorials/dynamic-1.html) system initialisation & threading with seL4_libs.
2. [Dynamic Libraries 2](https://docs.sel4.systems/Tutorials/dynamic-2.html) IPC with seL4_libs.
3. [Dynamic Libraries 3](https://docs.sel4.systems/Tutorials/dynamic-3.html) process management with seL4_libs.
4. [Dynamic Libraries 4](https://docs.sel4.systems/Tutorials/dynamic-4.html) timers and timeouts with seL4_libs.

- The slide presentations to guide you through the tutorials are in the following files:
  - `projects/sel4-tutorials/docs/seL4-Overview.pdf`: This is an overview of the design and thoughts behind seL4, and we strongly recommend you read it before starting the tutorials.
  - `projects/sel4-tutorials/docs/seL4-APILib-details.pdf`: This is the actual tutorial.

### CAmkES tutorials

These tutorials get you started with our component system CAmkES, which is allows you to configure static systems through components. CAmkES generates the glue code for interacting with seL4 and is designed for building high-assurance, static systems.

These tutorials work similarly to the SEL4 tutorials in that they are guided by a slide presentation. There are half-completed sample applications, with a set of slides giving instructions, with TASK challenges once again. There are also completed sample solutions.

1. [Camkes](https://docs.sel4.systems/Tutorials/hello-camkes-0.html) an introduction to Camkes concepts.
2. [Camkes 1](https://docs.sel4.systems/Tutorials/hello-camkes-1.html) an introduction to Camkes concepts.
3. [Camkes 2](https://docs.sel4.systems/Tutorials/hello-camkes-2.html) an introduction to Camkes concepts.
4. [Camkes 3](https://docs.sel4.systems/Tutorials/hello-camkes-timer.html) introduce Camkes hardware components.

- The slide presentations to guide you through the tutorials are in this file: `projects/sel4-tutorials/docs/CAmkESTutorial.pdf`.

### Virtual machines

These tutorials show how to run Linux guests on Camkes and how to communicate between them. Currently these tutorials are for x86 only.

1. [Camkes VM Linux](https://docs.sel4.systems/Tutorials/camkes-vm-linux.html) using Linux as a guest in the Camkes VM.
2. [Camkes Cross-VM communication](https://docs.sel4.systems/Tutorials/camkes-vm-crossvm.html) walkthrough of adding communication between Linux guests in separate VMs.

### MCS extensions

The MCS extensions are upcoming API changes to seL4.

1. [MCS](https://docs.sel4.systems/Tutorials/mcs.html) an introduction to the seL4 MCS extensions.
