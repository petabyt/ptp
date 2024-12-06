# What is PTP?

Picture Transfer Protocol was standardized in [2005](https://www.iso.org/standard/37445.html#:~:text=ISO%2015740%3A2005%20provides%20a,image%20storage%20and%20display%20devices.) and designed to be a generic
vendor-extensible image and file transfer protocol for digital cameras and printers.

MTP is a protocol based on PTP. It describes all of PTP, and adds a lot more features and functionality, and generally considered the most modern version of PTP.
https://www.usb.org/document-library/media-transfer-protocol-v11-spec-and-mtp-v11-adopters-agreement

Rather than exposing a filesystem, PTP exposes a generic API for downloading and listing *objects*. An 'object' can be anything, but in most cases it represents a file or folder on the device's storage. 
The API was designed this way because image transfer solutions at the time were proprietary and difficult to write drivers for on Linux. Thus, PTP was [drafted](https://people.ece.cornell.edu/land/courses/ece4760/FinalProjects/f2012/jmv87/site/files/PTP%20Protocol.pdf)
to create a single standard that vendors could build upon while still offering a standard file transfer protocol.

# How does it work?

PTP starts with the *initiator*. This is the device that issues commands to the *responder*. The initiator can issue commands such as `OpenSession` or `GetDeviceInfo`, and the responder
will reply back with a a return code, parameters, payload, etc.

Let's look at a very simple transaction. Here is the first packet sent from the initiator, after a USB connection is established:
```
.int 0xc+4
.short 0x2
.short 0x1002
.short 1
.int 1
.int 0x0
```

```
10 00 00 00
01 00
02 10
01 00 00 00
00 00 00 00 
```

This might be easier to look at if we encode it as a C struct.
```
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

A *session* is a connection between the initator and responder. Everything in a session including state, transaction IDs, object IDs, and descriptors are specific to a single session.
The only command allowed outside a session is `GetDeviceInfo`. PTP is designed this way to allow *multiple concurrent sessions* from different initiators. Sadly, it doesn't seem like any PTP device
supports this feature.

- Since this packet is sent in the *request phase*, the `type` field for this packet is `PTP_PACKET_TYPE_COMMAND` or `1`.
- A rule for `PtpOpenSession`: parameter 0 must contain the `SessionID`, a unique ID identifying a session. This can be whatever we want.
- Note that the `length` field is `16`. The minimium size of a request packet is `12` and the first parameter adds 4 bytes.
There can be up to 5 parameters in a packet.
- A rule for for `PtpOpenSession`: `transaction_id` will always be 0. The transaction ID is maintained by the initiator and incremented by `1` after every completed transaction.

PTP transactions consist of *phases*. A typical transaction will consist of a *request phase*, an (optional) *data phase*, and a *response phase*. The operation phase will always be a packet
sent from the *initiator*. Depending on the operation, the data phase can either be sent from the initiator or responder. For example, an operation like `GetDeviceInfo` will consist of data sent
from the responder, while an operation like `SetDevicePropValue` will have a data sent from the initiator. Note that *the initiator and responder cannot both send data*.

Since `PtpOpenSession` doesn't consist of any data phase, the response phase is next, with a `PTP_PACKET_TYPE_RESPONSE` packet (`3`) issued from the responder:
```
0c 00 00 00
03 00
01 20
01 00 00 00
```
- Here we see the `code` field is `0x2001` or `PTP_RC_OK`. Keep in mind all PTP codes start with a prefix in the higher 16 bits. For response codes (RC) it's `0x20`, for operation codes (OC) it's `0x10`, etc.
- Since this is packet is part of the same transaction, the transaction ID is the same.

Now, the transaction is over, and we will increment the client's internal transaction id counter before issuing the next transaction.

Another important thing to take note of is that even though all packets are labeled with a transaction ID, *a transaction must be fully completed before others are issued* (in each session). And if a
transaction ID changes in a packet before a transaction is finished, something is wrong.

# USB
A PTP/USB device should have at least two endpoints - one for sending data (`0x0`), and the other for receiving (`0x80`).
There may also be an interrupt endpoint (`0x81`).

## Sending data
Sending data is also a simple operation. **You cannot and receive data packets in a single transaction.** Some implementations may support it but it's *nonstandard behavior*.

After sending your command packet, send a data packet. For example:
Command packet:
- `length`: `16`
- `type`: `1`
- `code`: `0x1016` (`PTP_OC_SetDevicePropValue`)
- `transaction`: `current_transaction_id`
- `params[0]`: `0x5005` (`PTP_PC_WhiteBalance`)

Data packet:
- `length`: `14`
- `type`: `2`
- `code`: `0x1016` (`PTP_OC_SetDevicePropValue`)
- `transaction`: `current_transaction_id`
- `uint16_t` at offset `12`: `0x0001` (`Manual`)

... expect a response packet ...

## To issue a command to the device
1. Send a *command packet* with up to 5 parameters
2. An optional data packet (the data phase)
3. A response packet(s) is recieved from the camera.

For data to be sent to the camera, a data packet can be sent following  
the command packet. The camera should know when to expect this.  

- Each packet sent to the camera has a unique transaction ID (see PtpBulkContainer.transaction)
- The operation code (OC) is an ID for each command, and determines how the camera will expect and send back data.
- In a response packet, the response code (RC) is placed in the PtpBulkContainer.code field
