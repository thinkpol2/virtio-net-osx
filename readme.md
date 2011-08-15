# Virtio-Net for Mac OS X

## Latest release

I have released version 0.9.2 (the third beta for the 1.0), which includes support
for message signaled interrupts (they are not available on VirtualBox unfortunately),
and offloaded checksumming and TCP segmentation for IPv4. Transmit speeds with TSO are
now on par with receive speeds and easily outperform the emulated Intel gigabit
adapter.

Binaries (and the installer) are in the bin/ directory.

## Summary

Some virtualisation software (I know of VirtualBox and Linux KVM/Qemu) implements
paravirtual hardware per the "virtio" specification. One type of virtio device
is the "virtio-net" ethernet adapter. Linux and Windows guest drivers exist for
it, but as far as I know, this is the first such driver for Mac OS X (10.5+).

Compared to the default emulated Intel gigabit device, the paravirtualised adapter
is approximately twice as fast at transmitting TCP data (with TSO), and about 4
times as fast at receiving.

Kernel debugging via gdb is now also supported by this driver. If the virtio-net
device is the primary network adapter in the system (and the driver is the first
network card driver to be loaded), you can attach gdb to an appropriately
configured crashed kernel. Sending the ACPI Shutdown signal in VirtualBox is
treated as a non-maskable interrupt (NMI) so if you specify that kernel debug
flag as part of the boot args, you can attach the debugger that way.

## Virtio and virtio-net

[*virtio*][virtio] is an open specification for virtualised "hardware" in
virtual machines. Historically, virtual machine developers have either emulated
popular real hardware devices in order to utilise existing drivers in operating
systems, or implemented their own virtualised devices with drivers for each
supported guest operating system. Neither approach is ideal. Emulating
real hardware in software is often unnecessarily inefficient: some of the
constraints of real hardware don't apply to virtualised hardware. VM-specific
virtualised devices can be fast, but require specific driver for each supported
guest operating system. Moreover, the VM developer usually maintains the drivers,
and the specs are often not published. This prevents development of drivers for
less popular guest operating systems.

An open specification presents an
opportunity for separating the responsibilities for implementing the virtual
hardware and the drivers, and also potentially allows for greater guest
portability across different virtualisation solution.

The virtio spec (Version 0.9 as of this writing) includes a specification for a
virtualised PCI ethernet network card. Implementations for such virtual hardware
are present in Linux' KVM virtualisation solution and also in newer versions of
VirtualBox. Drivers for guests exist for Linux (in the main tree) and for
Windows.

## Motivation

VirtualBox supports virtual machines running Mac OS X (when running
on Apple hardware), but so far I have not found any virtio drivers. Virtual machines
are great for testing and debugging kernel extensions; I have so far however been unable
to connect gdb to a Mac OS X kernel running inside a VirtualBox VM with
emulated "real" ethernet cards. This is an attempt to create a driver which
supports kernel debugging for the virtio network interface, in addition to
being a good choice for general VM networking. Performance has not been a
priority but seems to be pretty good so far nevertheless.

## Status

Receiving and transmitting packets works, the adapter is able to negotiate DHCP,
send and receive pings, handle TCP connections, etc. The driver appears to be
stable even when saturating the virtual network's bandwidth. Some benchmarks
are in the docs/ directory, although these predate TSO support, which has
almost doubled transmit speed in VirtualBox on my Core2Duo MacBook Air.

Startup and shutdown appear to work fine, as do disabling and re-enabling the
device in Network Preferences and changing the adapter's configuration on the
host side.

The driver detects link status changes and correctly communicates it to the
operating system. This means that if you untick "cable connected" in the VirtualBox GUI for the
network device, the adapter's dot in the guest's Network Preferences turns red,
and back to green/yellow when you tick it. If you change the adapter's "wiring"
(bridging, NAT, host-only, etc.) this is communicated as a brief status change
which triggers a DHCP renew, as you'd want.

The "hardware" offers various advanced features, depending on implementation.
Of those, this driver supports checksum offloading and automatic segmentation
(TSO) for TCP on IPv4.

Reassembly
of large packets, MAC address filtering/promiscuous mode, VLAN filtering, etc.
are not implemented. Support may be added at a later date (patches welcome!).

## Next Steps

We should probably gather network statistics.

Error handling should be double-checked, and correct operation on `disable()` and
subsequent re-`enable()` should be verified. Correct freeing of all resources
should also be verified.

Currently, some
data coming from the device is blindly trusted. This isn't a big deal - if
we can't trust the VM container, we can't trust anything at all. Still, it would
be nice to avoid kernel panics or infinite loops on buggy virtio device
implementations, or in case of driver bugs.

I don't know if you can run Mac OS X on any other virtual machine containers
that support virtio network adapters, buf if you can, it would be nice to know
if the driver works on those.

## Future refinements

Longer term, if we wish to support other virtio devices, the PCI device handling
should be separated out into a distinct class from the ethernet controller. This
could then take care of general feature negotiation, memory mapping, virtqueue
handling, etc. To illustrate, the I/O registry hierarchy currently implemented
by this driver is:

    IOPCIDevice -> eu_philjordan_virtio_net[IOEthernetController] -> IOEthernetInterface

Where the object on the left side of the arrow is the provider of the one on the
right. Under the proposed scheme, it would look something like this:

    IOPCIDevice -> VirtioPCIDriver -> VirtioNetController[IOEthernetController] -> IOEthernetInterface

Other types of virtio devices would likewise attach to the `VirtioPCIDriver`.

## Compiling

If you simply want to use the driver, just use the installer. For compiling it
yourself, this repository contains
an XCode 4 project with which the KEXT can be built in a single step. The KEXT
should work on versions 10.5 (Leopard) through 10.7 (Lion), but so far has only
been tested on Snow Leopard. Since XCode 4 only runs on Snow Leopard and up,
you'll need to create your own XCode 3 project if you want to compile it on
Leopard.

## License

I'm making the source code for this driver available under the [zLib license][zlib],
the [3-clause BSD license][bsd3] and the [MIT License][mit]. The `virtio_ring.h`
file is adapted from the virtio spec and 3-clause BSD licensed. I've added the
MIT license in the hope that this will help inclusion into VirtualBox proper.

[virtio]: http://ozlabs.org/~rusty/virtio-spec/

[bsd3]: http://www.opensource.org/licenses/BSD-3-Clause

[zlib]: http://www.opensource.org/licenses/zLib

[mit]: http://www.opensource.org/licenses/MIT
