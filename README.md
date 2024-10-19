# How does PTP work?

Terminology:
- initiator: the computer that sends commands (laptop, phone, PC)
- responder: the computer that responds to commands (the camera)
- phase: A packet that is sent from one device to another
- transaction: A complete operation between an initiator and responder

Overview:
- Each packet that is sent between a PTP device is a *phase*. This typicaly consists of a operation request phase,
a data phase, and a response phase. The data phase is optional.
- All commands must be completed before another one is initiated. If a complete transaction takes a few seconds, then
all other transactions have to be blocked until that one is completed.

# PTP/USB
MTP is a protocol based on PTP. It describes all of PTP, and adds a lot more features and functionality.
https://www.usb.org/document-library/media-transfer-protocol-v11-spec-and-mtp-v11-adopters-agreement

## Performing a basic transaction - OpenSession
A PTP/USB device should have at least two endpoints - one for sending data (`0x0`), and the other for receiving (`0x80`).
There may also be an interrupt endpoint (`0x81`).

Upon connecting to a device, you should send an `OpenSession` request. You can do this by sending a container packet:
```
struct PtpBulkContainer {
	uint32_t length;
	uint16_t type;
	uint16_t code;
	uint32_t transaction;
	uint32_t params[5];
};
```
For `OpenSession`, we will set this structure as follows:
- `length`: `16` (`12 + (4 * n_of_params)`)
- `type`: `1` (`PTP_PACKET_TYPE_COMMAND`)
- `code`: `0x1002` (`PTP_OC_OpenSession`)
- `transaction`: `0` (always)
- `params[0]`: `1`

A few notes:
- `params[0]` is the `SessionID`. It can be any number over 1.
- Only one session can be opened at a time.
- For `OpenSession` the `transaction` field should always be `0`. **This number must be incremented in after every complete transaction.**

Next, read the device's response from the `0x80` endpoint. You should always get at least `12` bytes.

You will either get:
- A data (`PTP_PACKET_TYPE_DATA`) packet followed by a response (`PTP_PACKET_TYPE_RESPONSE`) packet
- Just a response packet.

Both packets will be in the `struct PtpBulkContainer` format, and the type will be in the `type` field.  

If you get a data packet:
- The `code` field should be the same as the `code` field in the command packet.
- The payload will be at offset `12`. the `params` field is skipped in data packets.
- The payload size is `length - 12`.

When you get the response packet:
- Check the `code` field and make sure it's a valid response (`0x2001`)
- Response packets may have return values in the `params` field.

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
