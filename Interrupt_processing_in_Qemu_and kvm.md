Introduction
=============

I wrote this mainly because I have been wondering for a long time how interrupts *actually* work in a virtual machine. Especially, I have been puzzled about the following:

- How do the physical interrupts get routed to VMs, and what are lapic, x2apic, ioapic etc
- What happens with interrupts from devices that have been assigned to VMs
- Why does x2apic not route interrupts to all cores, and how to fix that



Introduction to the hardware concepts
-------------------------------------

The purpose of this hardware summary is to introduce the concepts that then appear in Qemu.

At a very high level, from the CPU point of view, data enters it from a peripheral device such as a network card through some external connectors. When the data is ready, the device indicates this by issuing an interrupt request (IRQ). 

In the beginning of time, and still in some low-end systems, things work this way.

Picture:

     +-----+
     |     |
     |     + interrupt pin 
     |     | 
     | CPU +
     |     +  data pins
     |     +
     |     |
     +-----+ 

This is a very simple system, with just 3 data pins and a single interrupt pin. So it can communicate with the outside world 8 different data values. The interrupt pin is used to tell the CPU that data is now available.

In a more complex example, we would replace the data pins by a bus, so we could connect more devices to it. 

     +-----+
     |     | IRQ lines           dev1        dev2
     |     +--------------------++-+++------++ |||
     |     +---------------------+-+++-------+ |||
     |     | data pins             |||         |||
     | CPU +-----------------------+++---------+||
     |     +------------------------++----------+|
     |     +-------------------------+-----------+
     |     |
     +-----+ 

There also needs be to more interrupt pins, since we need to identify which device sent the data.


In conclusion, since the number of direct connections to the CPU is limited, the peripherals connect to an IO bus. There are different types of IO buses with familiar names such as PCI, ISA, SCSI and USB, but a given system has only one system bus which is typically PCI. Other buses can be connected to the system bus by using bridges.

Let's first discuss the data part. In a more complicated system, the CPU no longer bothers to read the data from the data pins directly. A more high level concept is to map the device data to the memory that the CPU sees. This is known as memory mapped IO and the device configuration is often done this way. The CPU can both write to the device and read from the device this way.

If there is a lot of data that the CPU needs to copy to memory in order to manipulate it, it can set up a direct memory access for the devices. This means that data from the device is copied to the real memory. This obviously requires that someone knows where the data from a device should be copied to. The IOMMU unit handles this.

The interrupt handling gets more complicated. 

     +-----+-+-----+
     |     +-+ PIC |  IRQ lines  dev1        dev2
     |     +-+     +------------++-+++------++ |||
     |     +-+-----+-------------+-+++-------+ |||
     |     | data pins             |||         |||
     | CPU +-----------------------+++---------+||
     |     +------------------------++----------+|
     |     +-------------------------+-----------+
     |     |
     +-----+ 

To save work for the CPU, the IRQ lines (physical connectors) for the physical devices can terminate in a piece of hardware called Programmable Interrupt Controller (PIC). The PIC has data connections to the CPU and it is also connected to the INTR pin of the CPU, which is the modern way to issue interrupts to CPUs. The PIC monitors the IRQ lines and when something happens, it sets an interrupt vector corresponding to the IRQ on this data pins. It then issues an interrupt to the CPU which reads the interrupt vector and then knows the reason for the interrupt.

The PIC can disable certain interrupts while the CPU is doing something else, and then issue the interrupt when the CPU tells it is ok to do so. It can also mask interrupts so they are not issued at all.

This works in a uniprocessor system. In a multiprocessor system, we need an I/O Advanced PIC to receive all IRQs, and local APICs (lapics) for each CPU. The I/O APIC forwards the interrupts to the local APICs which then forward them to their CPUs. The IO APIC evolved into Extended APIC (xapic) and the x2apic. I use the term x2apic from now on for the IO APIC.

The x2apic has an Interrupt Redirection Table (IRT) which tells what to do with each IRQ: what is the corresponding interrupt vector, what the interrupt priority is, which CPU can handle it and how to choose the right CPU. The selection can be 
- static so the interrupt always goes to the same CPU or set of CPUs or to all CPUs
- dynamic so the interrupt goes to the CPU that is executing the lowest priority task.

How does the x2apic know which CPU is executing the lowest priority task? Task priority is an operating system concept, not a CPU feature. The host operating system is supposed to keep the x2apic up to date by writing to the task priority register (TPR) every time it does a task switch. If there are many CPUs with the same priority, the x2apic will select them in order. In practice, Linux sets the TPR to a fixed value and never changes it (see setup_local_APIC(void) in arch/x86/kernel/apic/apic.c).

On the CPU side, the Interrupt Descriptor Table (IDT) keeps track of the action that the CPU should take when it receives an interrupt. It maps the interrupt vector to the action. The register idtr tells where in memory the IDT is.

Enough for the HW concepts. Now to virtualization.


Introduction to Qemu/kvm virtualization
----------------------------------------

Qemu by itself is a hardware emulator that allows to run unmodified binary code for any architecture on a Qemu process on any other architecture (with some limitations, of course). It does this by emulating in software the whole hardware environment, including the CPU. This works but is quite slow.

Kvm improves the performance by allowing native code to run natively. In this case, there is almost no performance difference native execution speed and code running in kvm. The challenge in running VM code natively is to make sure that the code does not execute any instructions that it should not. This is quite difficult, since the VM is running an operating system that expects to manage the hardware, physical memory etc. 

The solution that kvm uses is relying on the Intel VT-x hardware virtualization functionality. It has a new processor mode for VM code, and new instructions for entering (VMENTRY) and exiting (VMEXIT) the VM mode. These transitions are quite expensive, although the cost has gone down with new hardware functionality. More about that later.

Kvm by itself is a kernel module and Qemu is a user space entity. Qemu is emulating most of the hardware, while kvm is running the CPU code and doing a limited number of high-performance tasks. Qemu communicates with kvm by using ioctl. The vCPU loop goes as follows:


Kvm vCPU loop

Qemu: 	userspace main loop ->                                                
	(ioctl(KVM_RUN))  
Kvm:	Kernel main loop ->
	(VMENTER)
Native:	Guest execution ->
	(VMEXIT)
Kvm:	Kernel exit handler ->
Qemu:	Userspace exit handler
(repeat)

Qemu starts first. It executes its main loop until it calls the ioctl(KVM_RUN). 

APIC emulation
	
	
Interrupt control in Intel processors
--------------------------------------

Control of External-Interrupts. 
VMX allows both host and guest control of external interrupts through the "external-interrupt exiting" VM execution control. 
With guest control (external-interrupt exiting set to 0), external-interrupts do not cause VM exits and the interrupt delivery is masked by the guest programmed RFLAGS.IF value.1 
With host control (external-interrupt exiting set to 1), external-interrupts causes VM exits and are not masked by RFLAGS.IF. 
The VMM can identify VM exits due to external interrupts by checking the exit-reason for an external-interrupt (value = 1).

With host control of external interrupts, the VMM (or the host OS in a hosted VMM model) manages the physical interrupt controllers in the platform and the interrupts generated through them. 
The VMM exposes software-emulated virtual interrupt controller devices (such as PIC and APIC) to each guest virtual machine instance.


The Intel 64 and IA-32 architectures use 8-bit vectors of which 244 (20H - FFH) are available for external interrupts. 
Vectors are used to select the appropriate entry in the interrupt descriptor table (IDT). 
VMX operation allows each guest to control its own IDT. 
Host vectors refer to vectors delivered by the platform to the processor during the interrupt acknowledgement cycle. 
Guest vectors refer to vectors programmed by a guest to select an entry in its guest IDT. 
Depending on the I/O resource management models supported by the VMM design, the guest vector space may or may not overlap with the underlying host vector space.

- Interrupts from virtual devices: Guest vector numbers for virtual interrupts delivered to guests on behalf of emulated virtual devices have no direct relation to the host vector numbers of interrupts from physical devices on which they are emulated. A guest-vector assigned for a virtual device by the guest operating environment is saved by the VMM and utilized when injecting virtual interrupts on behalf of the virtual device.

- Interrupts from assigned physical devices: Hardware support for I/O device assignment allows physical I/O devices in the host platform to be assigned (direct-mapped) to VMs. 
Guest vectors for interrupts from direct-mapped physical devices take up equivalent space from the host vector space, and require the VMM to perform host-vector to guest-vector mapping for interrupts.







Questions
---------

1)	Intel VT FlexPriority

Not mentioned in SDM

2)	EPT accessed and dirty bits
3)	Virtual Processor IDs (VPIDs)

Yes, this should work

4)	Multiqueue Networking for KVM

Yes in theory



References:
===========

Understanding the Linux Kernel
Daniel P. Bovet, Marco Cesati
Third Edition
O'Reilly


Enabling Optimized Interrupt/APIC Virtualization in KVM
Jun Nakajima
Intel Open Source Technology Center
November 8, 2012
Kvm Forum 2012


For AMD
---------

Next-generation Interrupt Virtualization for KVM
Jrg Rdel
August 2012

KVM Weather Report
Gleb Natapov
May 29, 2013


For Intel
----------

Intel(R) 64 and IA-32 Architectures Software Developer's Manual, Volume 3B
Chapter 29

