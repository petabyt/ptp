# How does PTP work?
MTP is a protocol based on PTP. It describes all of PTP, and adds a lot more features and functionality.
https://www.usb.org/document-library/media-transfer-protocol-v11-spec-and-mtp-v11-adopters-agreement

Terminology:
- initiator: the computer that sends commands (laptop, phone, PC)
- responder: the computer that responds to commands (the camera)
- phase: A packet that is sent from one device to another
- transaction: A complete operation between an initiator and responder

## Quick Overview of PTP Standard
- Each packet that is sent between a PTP device is a *phase*. This typicaly consists of a operation request phase,
a data phase, and a response phase. The data phase is optional.
- All commands must be completed before another one is initiated. If a complete transaction takes a few seconds, then
all other transactions have to be blocked until that one is completed.

### To issue a command to the device
1. Send a *command packet* with up to 5 parameters
2. An optional data packet (the data phase)
3. A response packet(s) is recieved from the camera.

For data to be sent to the camera, a data packet can be sent following  
the command packet. The camera should know when to expect this.  

- Each packet sent to the camera has a unique transaction ID (see PtpBulkContainer.transaction)
- The operation code (OC) is an ID for each command, and determines how the camera will expect and send back data.
- In a response packet, the response code (RC) is placed in the PtpBulkContainer.code field
