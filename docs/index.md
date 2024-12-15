# What is PTP?
Picture Transfer Protocol was standardized in [2005](https://www.iso.org/standard/37445.html#:~:text=ISO%2015740%3A2005%20provides%20a,image%20storage%20and%20display%20devices.) and designed to be a generic
vendor-extensible image and file transfer protocol for digital cameras and printers.

MTP is a superset of PTP which adds additional features and improvements over the 2005 standard. The [MTP spec](https://www.usb.org/document-library/media-transfer-protocol-v11-spec-and-mtp-v11-adopters-agreement)
is freely available and well-written, and is great documentation for anybody who wants to write a client.

Unlike a mass storage device, PTP doesn't expose a filesystem.
Instead, it exposes a generic API for listing, downloading, and uploading *objects*. An 'object' can be anything, but in most cases it represents a file or folder on the device's storage medium. 

There's a few reasons why PTP was designed this way the time:

- Image transfer solutions at the time were proprietary and difficult for Linux developers to write drivers for
- Abstracting away the filesystem allowed the camera to store images in any way it liked (even without a filesystem)
- Removed the risk of filesystem corruption
- Allows the camera to safely modify objects while the client is downloading/uploading data
- Manufacturers don't just want a image transfer protocol, they want to build upon it and add custom functionality (such as liveview, remote capture, etc)

Thus, in 2002 PTP was [drafted](https://people.ece.cornell.edu/land/courses/ece4760/FinalProjects/f2012/jmv87/site/files/PTP%20Protocol.pdf)
to create a single standardized protocol that vendors could build upon while still offering an easy to implement file transfer solution.

# How does it work?
PTP starts with the *initiator*, or the client. This is the device that issues commands to the *responder*, which can be a camera, printer, or phone.
The initiator can issue commands such as `OpenSession` or `GetDeviceInfo`, and the responder will reply back with a a return code, parameters, payload, etc.

Another important thing to understand is that PTP transactions consist of **phases**. A typical transaction will consist of a *request phase*, an (optional) *data phase*, and a *response phase*.

- The *operation phase* will always consist of a single request packet sent from the *initiator*.
- The *data phase* can consist of a data packet sent from either the initiator or responder, depending on the operation.
- The *response phase* will always consist of a single response packet.

For example, an operation like `GetDeviceInfo` will consist of data received from the responder, while an operation like `SetDevicePropValue` will have data sent from the initiator.

Note that *the initiator and responder cannot both send data in the same transaction*.

<!--
```
//temp
.int 0xc+4
.short 0x2
.short 0x1002
.short 1
.int 1
.int 0x0
```
-->
## `PtpOpenSession`
Let's look at a very simple transaction from the viewpoint of the initiator. Here is the first packet we will send to the responder, after a USB connection is established:
```
10 00 00 00
01 00
02 10
01 00 00 00
00 00 00 00 
```

*Keep in mind that PTP data structures are encoded as little endian, and fields may not be aligned.*

This might be easier to look at if we encode it as a C struct.
```c
struct PtpContainer {
	uint32_t length;
	uint16_t type;
	uint16_t code;
	uint32_t transaction_id;
	uint32_t params[5];
}command = {
	.length = 16,
	.type = 1,
	.code = 0x1002,
	.transaction_id = 0,
	.params = {
		0
	}
};
```

Generally the first request sent to the device will be `PtpOpenSession` or `0x1002`. This opens a *session* with the responder.

A **session** is a connection between the initator and responder. Each session has it's own state, transaction IDs, object IDs, and descriptors.
The only command allowed outside a session is `GetDeviceInfo`. PTP is designed this way to allow **multiple concurrent sessions** from different initiators. Sadly, it doesn't seem like any PTP device
supports this feature.

- Since this packet is sent in the *request phase*, the `type` field for this packet is `PTP_PACKET_TYPE_COMMAND` or `1`.
- A rule for `PtpOpenSession`: parameter 0 must contain the `SessionID`, a unique ID identifying a session. This can be whatever we want.
- Note that the `length` field is `16`. The minimium size of a request packet is `12` and the first parameter adds 4 bytes.
There can be up to 5 parameters in a command packet.
- A rule for for `PtpOpenSession`: `transaction_id` will always be 0. The transaction ID is maintained by the initiator and incremented by `1` after every completed transaction.

Since `PtpOpenSession` doesn't have a data phase, the response phase is next, with a `PTP_PACKET_TYPE_RESPONSE` packet (`3`) issued from the responder:
```text
0c 00 00 00
03 00
01 20
01 00 00 00
```
- Here we see the `code` field is `0x2001` or `PTP_RC_OK`. Keep in mind all PTP codes start with a prefix in the higher 16 bits. For response codes (RC) it's `0x20`, for operation codes (OC) it's `0x10`, etc.
- Since this is packet is part of the same transaction, the transaction ID is the same.

Now, the transaction is over, and we will increment the client's internal transaction ID counter before issuing the next transaction.

Another important thing to take note of is that even though all packets are labeled with a transaction ID, **a transaction must be fully completed before others are issued** (in each session). You cannot issue
another transaction while one is in progress. If the transaction ID changes before the transaction is over, then something is wrong.

## `PtpGetDeviceInfo`
Next, let's have our client send a `PtpGetDeviceInfo` request. This operation returns a data structure describing the device we are communicating with.

Here is the request packet we will send to the responder:
```text
0c 00 00 00
01 00
01 10
02 00 00 00
```
The transaction id field is now `2`, and the code field is now set to `0x1001`.

When we read data from the responder, we should get both a data packet (`PTP_PACKET_TYPE_DATA`) and a response packet (`PTP_PACKET_TYPE_RESPONSE`), in that order.

The data packet is big, but is easy to decode:
```text
00000000  0b 02 00 00 02 00 01 10  01 00 00 00 64 00 06 00  |............d...|
00000010  00 00 64 00 00 00 00 a7  00 00 00 14 10 15 10 16  |..d.............|
00000020  10 01 10 02 10 03 10 06  10 04 10 01 91 05 10 02  |................|
00000030  91 07 10 08 10 03 91 09  10 04 91 0a 10 1b 10 07  |................|
00000040  91 0c 10 0d 10 0b 10 05  91 0f 10 06 91 10 91 27  |...............'|
00000050  91 0b 91 08 91 09 91 0c  91 0e 91 0f 91 15 91 14  |................|
00000060  91 13 91 16 91 17 91 20  91 f0 91 18 91 21 91 f1  |....... .....!..|
00000070  91 1d 91 0a 91 1b 91 1c  91 1e 91 1a 91 53 91 54  |.............S.T|
00000080  91 60 91 55 91 57 91 58  91 59 91 5a 91 1f 91 fe  |. .U.W.X.Y.Z....|
00000090  91 ff 91 28 91 29 91 2d  91 2e 91 2f 91 2c 91 30  |...(.).-.../.,.0|
000000a0  91 31 91 32 91 33 91 34  91 2b 91 35 91 36 91 37  |.1.2.3.4.+.5.6.7|
000000b0  91 38 91 39 91 3a 91 3b  91 3c 91 da 91 db 91 dc  |.8.9.:.;.<......|
000000c0  91 dd 91 de 91 d8 91 d9  91 d7 91 d5 91 2f 90 41  |............./.A|
000000d0  91 42 91 43 91 3f 91 33  90 68 90 69 90 6a 90 6b  |.B.C.?.3.h.i.j.k|
000000e0  90 6c 90 6d 90 6e 90 6f  90 3d 91 80 91 81 91 82  |.l.m.n.o.=......|
000000f0  91 83 91 84 91 85 91 40  91 01 98 02 98 03 98 04  |.......@........|
00000100  98 05 98 c0 91 c1 91 c2  91 c3 91 c4 91 c5 91 c6  |................|
00000110  91 c7 91 c8 91 c9 91 ca  91 cb 91 cc 91 ce 91 cf  |................|
00000120  91 d0 91 d1 91 d2 91 e1  91 e2 91 e3 91 e4 91 e5  |................|
00000130  91 e6 91 e7 91 e8 91 e9  91 ea 91 eb 91 ec 91 ed  |................|
00000140  91 ee 91 ef 91 f8 91 f9  91 f2 91 f3 91 f4 91 f7  |................|
00000150  91 22 91 23 91 24 91 f5  91 f6 91 52 90 53 90 57  |.".#.$.....R.S.W|
00000160  90 58 90 59 90 5a 90 5f  90 07 00 00 00 09 40 04  |.X.Y.Z._......@.|
00000170  40 05 40 03 40 02 40 07  40 01 c1 05 00 00 00 02  |@.@.@.@.@.......|
00000180  d4 07 d4 06 d4 03 d3 01  50 01 00 00 00 01 38 0c  |........P.....8.|
00000190  00 00 00 01 30 02 30 06  30 0a 30 08 30 01 38 01  |....0.0.0.0.0.8.|
000001a0  b1 03 b1 02 bf 00 38 04  b1 05 b1 0b 43 00 61 00  |......8.....C.a.|
000001b0  6e 00 6f 00 6e 00 20 00  49 00 6e 00 63 00 2e 00  |n.o.n. .I.n.c...|
000001c0  00 00 13 43 00 61 00 6e  00 6f 00 6e 00 20 00 45  |...C.a.n.o.n. .E|
000001d0  00 4f 00 53 00 20 00 52  00 65 00 62 00 65 00 6c  |.O.S. .R.e.b.e.l|
000001e0  00 20 00 54 00 36 00 00  00 08 33 00 2d 00 31 00  |. .T.6....3.-.1.|
000001f0  2e 00 32 00 2e 00 30 00  00 00 08 38 00 32 00 38  |..2...0....8.2.8|
00000200  00 61 00 66 00 35 00 36  00 00 00                 |.a.f.5.6...|
0000020b
```

Data packets have no parameters, so the payload always starts at offset `12`.

The beginning of the payload starts with 3 fields:
```
uint16_t standard_version;
uint32_t vendor_ext_id;
uint16_t version;
```
What follows next are 6 `PTP_TC_UINT16ARRAY` arrays describing what codes the device supports. Parsing these is as simple as
reading the first 4 bytes as the array length (`uint32_t`), reading the next `length * 2` bytes as `uint16_t`.

Next are 4 strings. Almost all strings in PTP are encoded as *wide strings*. The first byte is read as the length of the string (in chars),
and the rest are a series of wide chars (`int16_t`) with a null terminator. The length field includes the null terminator.

## Sending data to the responder

For example, sending a `SetDevicePropValue` request:
```text
10 00 00 00
01 00
16 10
03 00 00 00
05 50 00 00
```
The first parameter is `0x5005`, or `PTP_PC_WhiteBalance`

As described in the MTP spec, `PTP_PC_WhiteBalance` is a `UINT16` data type.
So the data packet in this case will be 2 bytes:
```text
0e 00 00 00
02 00
16 10
03 00 00 00
01 00
```
This sets `PTP_PC_WhiteBalance` to `0x0001`, or 'Manual'.
After this, we expect a response packet from the responder.

# What's next?
The packet structure described in this document is for PTP/USB only. PTP/IP has a totally different packet structure and way of sending packets.
Thankfully, I have started to document it [here](ip).

# Notes on USB
A PTP/USB device should have at least two endpoints - one for sending data and the other for receiving.
There may also be an interrupt endpoint. This is a read only endpoint that the client may poll to get event codes.

The class for PTP devices should be `0x06` for Still Imaging devices, as described by the [USB forum](https://www.usb.org/defined-class-codes)

# vcam
[vcam](https://github.com/petabyt/vcam) is an responder implementation of PTP, MTP, and other proprietary PTP extensions that can be used with unmodified PTP clients.
This may be useful if you want test yours.
