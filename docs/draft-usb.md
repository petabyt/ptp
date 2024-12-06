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
