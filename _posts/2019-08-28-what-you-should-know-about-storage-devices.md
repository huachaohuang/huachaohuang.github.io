---
layout: post
title: What You Should Know About Storage Devices
---

I have been working on various storage devices for years.
However, if you ask me about the differences between SAS SSD, SATA SSD, and NVMe SSD, well, I can't tell. I just know that their performance varies.

I tried to look into these storage devices from time to time over the years.
I have learned more and more terms like IDE, SCSI, PCIe, but I don't know how to put these pieces together.
Then I get interested in the block device layer on Linux these days, which requires some storage devices specific knowledge to understand.
So I decide that this is a good chance to take the time and solve the puzzle.

I spent the past few days digging the web and built a *Storage Architecture Model*.
This model serves as a framework to put storage devices related knowledge together, in the perspective of a database developer.
However, I build this model based on my understanding, which is limited and may not be precise.
So, I will be very grateful if you can correct me or provide more insight.

## Background

Before talking about the model, I need to give you some background knowledge first.

Let's say you have a CPU and a disk.
You want to let the CPU save your data to the disk, what do you need to do?
Well, you need to connect the CPU with the disk so that they can communicate with each other at least.
Maybe you can find some wires to tie them together, but that's too hardcore. Besides, do you really know how to do that?

![Point-to-point Construction](https://upload.wikimedia.org/wikipedia/commons/5/5a/Motorolagoldenviewchassis.jpg)

Forget it, let's use a [motherboard](https://en.wikipedia.org/wiki/Motherboard) instead, which provides a CPU slot and some expansion slots for peripheral devices.

![Motherboard](https://upload.wikimedia.org/wikipedia/commons/thumb/d/de/386DX40_MB_Jaguar_V.jpg/800px-386DX40_MB_Jaguar_V.jpg)

OK, now you may think that you can just insert your CPU and your disk to those slots, and then everything works, like magic.
But things are never as simple as that.
Do you know how many different kinds of CPUs, disks, and motherboards on the market?
How can you know if your CPU, your disk, and your motherboard fit together?

Yes, we need to make some standards for them. Here is one thing I know about standards:

![Standards](https://imgs.xkcd.com/comics/standards.png)

Right, there are a lot of standards out there but don't worry, we don't need to know all of them.
According to my work experience on storage devices, we only need to pay attention to three standard families: SCSI, ATA, and PCI.
I split each standard into three layers according to my understanding: Command Layer, Control Layer, and Transport Layer.
My storage architecture model also includes a Storage Media Layer to complete the hardware-level description of storage devices.
I'm going to talk about these layers one by one next, but I need to put an overview of these standards ahead so that I don't need to explain these terms elsewhere.

### Standards

#### SCSI

- [Small Computer System Interface (SCSI)](https://en.wikipedia.org/wiki/SCSI)
- [SCSI Parallel Interface (SPI)](https://en.wikipedia.org/wiki/Parallel_SCSI)
- [Serial Attached SCSI (SAS)](https://en.wikipedia.org/wiki/Serial_Attached_SCSI)

SCSI Standards Architecture:

![SCSI Standards Architecture](http://www.t10.org/scsi-3.jpg)

You can find more about SCSI standards from [INCITS T10 Technical Committee](http://www.t10.org/).

#### ATA

- [AT Atachment (ATA)](https://en.wikipedia.org/wiki/Parallel_ATA)
- [Parallel AT Atachment (PATA)](https://en.wikipedia.org/wiki/Parallel_ATA)
- [Serial AT Attachment (SATA)](https://en.wikipedia.org/wiki/Serial_ATA)
- [Advanced Host Controller Interface (AHCI)](https://en.wikipedia.org/wiki/Advanced_Host_Controller_Interface)

ATA Standards Architecture:

![ATA Standards Architecture](https://www.iso.org/obp/graphics/std//iso_std_iso-iec_17760-102_ed-1_v1_en/fig_1.png)

You can find more about ATA standards from [INCITS T13 Technical Committee](http://t13.org/).

#### PCI

- [Peripheral Component Interconnect (PCI)](https://en.wikipedia.org/wiki/Conventional_PCI)
- [PCI Express (PCIe)](https://en.wikipedia.org/wiki/PCI_Express)
- [Non-Volatile Memory (NVM)](https://en.wikipedia.org/wiki/Non-volatile_memory)
- [NVM Express (NVMe)](https://en.wikipedia.org/wiki/NVM_Express)

You can find more about PCI standards from [PCI-SIG](https://pcisig.com/).

## Storage Media

Storage media are the physical materials to store data.
We have machanical media (e.g. [punched card](https://en.wikipedia.org/wiki/Punched_card)), magnetic media (e.g. [magnetic tape](https://en.wikipedia.org/wiki/Magnetic_tape)), and electrical media (e.g. [flash memory](https://en.wikipedia.org/wiki/Flash_memory)).
You can check this video about [How Flash Memory Works](https://www.youtube.com/watch?v=s7JLXs5es7I).

[Hard Disk Drive (HDD)](https://en.wikipedia.org/wiki/Hard_disk_drive) and [Solid-State Drive (SSD)](https://en.wikipedia.org/wiki/Solid-state_drive) are two common types of storage devices.

HDD uses magnetic disks as its storage media and contains some electro-mechanical components.
You can check this video about [How HDD Works](https://www.youtube.com/watch?v=Ep-yM894mQQ).

SSD uses electrical media like NAND Flash and [3D XPoint](https://en.wikipedia.org/wiki/3D_XPoint) and contains only electronic circuits.
You can check this post about [How SSD Works](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/).

## Command Layer

Command Layer defines the command set host systems use to control storage devices.

SCSI, ATA, and NVMe standards contain specifications about this layer.

SCSI splits its command set standards into two parts.
The first part is SCSI Primary Commands (SPC), which defines commands that apply to all SCSI devices.
The second part is device type specific command sets like SCSI Block Commands (SBC), which defines commands specific to SCSI block devices.
So, for example, a SCSI block device needs to support both SPC and SBC.

ATA Command Set (ACS) defines commands to control ATA storage devices, including PATA and SATA devices.
During years of evolution, ATA command set and SCSI command set share a lot of common, especially for block devices.
SCSI also defines a SCSI/ATA Translation (SAT) standard to translate some SCSI commands to ATA commands.

NVMe command set defines the commands to control NVM devices.
Before NVMe came up, people used SCSI and ATA command sets to control SSDs.
But SCSI and ATA were primarily designed for HDDs and they can't fully utilize the potential of SSDs.
NVMe has been designed to capitalize on the low latency and internal parallelism of SSDs and has become the de facto standard for SSDs now.

## Control Layer

People use different terms in different contexts to describe how host systems control storage devices.

In the software level, this layer acts as a programming interface.
It might be referred to as:
- Register Interface: because it interacts with devices using registers
- Host Controller Interface (HCI): because it specifies the operation of host controllers

In the hardware level, this layer exists as a chip integrated on an [expansion card](https://en.wikipedia.org/wiki/Expansion_card) or a motherboard (with a chipset called [Southbridge](https://en.wikipedia.org/wiki/Southbridge_(computing)), [I/O Controller Hub](https://en.wikipedia.org/wiki/I/O_Controller_Hub) or [Platform Controller Hub](https://en.wikipedia.org/wiki/Platform_Controller_Hub), whatever).
It might be referred to as:
- I/O Controller: because it controls I/O devices
- Host Controller: because it controls devices for host systems
- [Host Bus Adapter (HBA)](https://en.wikipedia.org/wiki/Host_adapter): because it adapts devices to host buses.

These terms can be confusing, so I decide to name it *Control Layer* here based on what it does.

To understand this layer, you need to get some basic knowledge about CPU first.
You can check this video to [See How a CPU Works](https://www.youtube.com/watch?v=cNN_tTXABUA).
A CPU can't communicate with other devices directly because they don't share the same physical and electrical characteristics (I will talk about that later).
Instead, a CPU needs to access some specific addresses (or registers) binding to a controller and then lets the controller deal with other devices for it.
So, this layer defines the interface between a CPU and a controller, for example, specifies how to put a command to a command queue.

ATA, AHCI, and NVMe standards contain specifications about this layer.

ATA defines the interface to interact with PATA devices over parallel buses.
SATA devices use AHCI instead, which works as a PCI device.
NVMe defines the interface to interact with NVM devices over PCIe buses.
Again, NVMe has been designed from the group up for SSDs since AHCI can't deliver the optimal performance with SSDs.
Check this [comparison between NVMe and AHCI](https://en.wikipedia.org/wiki/NVM_Express#Comparison_with_AHCI).
What about SCSI devices, you may ask.
Well, I can't find any public SCSI standard about this layer, so I think vendors don't come up a common standard here.
But I can find some vendor-specific drivers on Linux that implement the interfaces for SCSI devices.

I will talk about different buses next.

## Transport Layer

Most standards describe how storage devices communicate with each other in three layers:
- Physical Layer: defines the physical interconnect (e.g. cables, connectors) and electrical characteristics
- Data Link Layer: provides a reliable mechanism to exchange data, including error detection and frame transmission, etc
- Transport Layer: specifies the protocol (interaction) between two devices, including frame formats and how frames are processed, etc

I redefine the *Transport Layer* here to include all the three layers above.
Because as an upper-level software developer, I don't deal with those layers as long as these devices can talk to each other.

In computer architecture, people use the term [bus](https://en.wikipedia.org/wiki/Bus_(computing)) to represent a communication system that transfers data between components.
There are two kinds of buses: parallel and serial.
Parallel buses transfer multiple bits at a time while serial buses transfer only one bit at a time.
However, parallel buses are slower than serial buses because they operate at a much lower frequency.

SCSI, ATA, and PCI are three standard families that contain specifications about this layer.
Each of them defines a parallel bus and a serial bus. Let's put them into a table here:

|          | SCSI                          | ATA                 | PCI                |
|----------|-------------------------------|---------------------|--------------------|
| Parallel | SCSI Parallel Interface (SPI) | Parallel ATA (PATA) | Conventional PCI   |
| Serial   | Serial Attached SCSI (SAS)    | Serial ATA (SATA)   | PCI Express (PCIe) |

Nowadays, parallel interfaces are mostly replaced by serial interfaces because of their better performance.
These buses have evolved a few versions, providing higher and higher speed to suit the evolution of other devices.

## Storage Architecture Model

Now we can put all the layers together and build our storage architecture model.

![Storage Architecture Model](/images/storage_architecture_model.png)

That's it, clean and compact. Next time you ask me what a SATA HDD or an NVMe SSD is, I can tell you loud and clear.

SATA HDD is a storage device, it accepts ATA commands, works with an AHCI controller, transfers data over a SATA bus, and stores data with magnetic disks.

NVMe SSD is a storage device, it accepts NVMe commands, works with an NVMe controller, transfers data over a PCIe bus, and stores data with NAND Flash or other electrical media.

But there is one more thing.

If you try to buy an NVMe SSD, you may find something like "NVMe PCIe M.2 SSD".

![NVMe PCIe M.2 SSD](/images/nvme_pcie_m2_ssd.jpg)

If you follow me all along here, I think you already know what NVMe, PCIe, and SSD mean, but what is M.2?
[M.2](https://en.wikipedia.org/wiki/M.2) is a connector standard, it defines how to connect a device to a motherboard, like the shape of those golden pins on the right of the picture above.

There are a lot of connector standards, for example, [U.2](https://en.wikipedia.org/wiki/U.2), mSAS, and [mSATA](https://en.wikipedia.org/wiki/Serial_ATA#Mini-SATA_(mSATA)).
I don't mention them until now to keep them out of the model so that they don't muddy the water in the first place.
You don't need to pay much attention to them unless you are trying to buy an SSD that fits your motherboard.
Again, as a software developer, who cares about the different shapes of these devices as long as they can be connected to the motherboard.
