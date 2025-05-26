# PTP/IP
PTP/IP is the variant of PTP designed to be used on TCP.

All packet types start with a `length` and `type` field. The layout of the rest of the fields
depends on `type`.

A standard request to the responder is as follows:

## 1. Send Command request
```
struct PtpIpBulkContainer {
	uint32_t length;
	uint32_t type;
	uint32_t data_phase;
	uint16_t code;
	uint32_t transaction;
	uint32_t params[5];	
};
```
Rules:

- Set `length` to `18 + (4 * number_of_params)`
- Set `type` to `PTPIP_COMMAND_REQUEST` (`0x6`)
- Set the `data_phase` field to 2 if you are sending a data payload. Set to 1 if otherwise.
- Set code to a PTP_OC_ opcode.
- Set the `transaction` to the transaction ID for this operation.

## 2. Send Data Start packet
```
struct PtpIpStartDataPacket {
	uint32_t length;
	uint32_t type;
	uint32_t transaction;
	uint64_t payload_length;
};
```
Rules:

- Set `length` to `20`.
- Set `type` to `PTPIP_DATA_PACKET_START` (`0x9`)
- Set `transaction` to the transaction ID for this operation.
- Set `data_phase_length` to the size of the payload in the following data packet.

## 3. Send Data End Packet
```
struct PtpIpEndDataPacket {
	uint32_t length;
	uint32_t type;
	uint32_t transaction;
	// payload data goes here
};
```
Rules:

- Set length to `12 + payload_length`
- Set type to `PTPIP_DATA_PACKET_END` (`0xc`)
- Set `data_phase_length` to the size of the payload in the following data packet.
- Copy the payload that will be sent to the device at offset `12`.

# 4. Responses
For an operation, you may get a data payload as a response (R->I) 

The response to an operation will either be:
- `PTPIP_DATA_PACKET_START` packet followed by `PTPIP_DATA_PACKET_END`, and finally a `PTPIP_COMMAND_RESPONSE` packet
- Just a `PTPIP_COMMAND_RESPONSE` packet

## Response Container
```
struct PtpIpResponseContainer {
	uint32_t length;
	uint32_t type;
	uint16_t code;
	uint32_t transaction;
	uint32_t params[5];
};
```
