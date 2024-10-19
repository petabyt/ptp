# CHDK PTP Extension
- Implementation: https://github.com/petabyt/chdk/blob/23c1f3f04fa1c7d483083d403818972a739c6dfd/core/ptp.c#L443
- Opcode: `0x9999`

All functionality is switched from a code in the first parameter: https://github.com/petabyt/chdk/blob/master/trunk/core/ptp.c#L33C1-L33C1

- [ptp.c](https://github.com/petabyt/chdk/blob/master/trunk/core/ptp.c)
- [ptp.h](https://github.com/petabyt/chdk/blob/master/trunk/core/ptp.h)

```
enum ptp_chdk_command {
  PTP_CHDK_Version = 0,     // return param1 is major version number
                            // return param2 is minor version number
  PTP_CHDK_GetMemory,       // param2 is base address (direct may fail on MMIO etc. Use buffered for those)
                            // param3 is size (in bytes)
                            // param4 is options: 0 read directly, 1 buffer. Other values reserved
                            // return data is memory block
  PTP_CHDK_SetMemory,       // param2 is address
                            // param3 is size (in bytes)
                            // data is new memory block
  PTP_CHDK_CallFunction,    // data is array of function pointer and 32 bit int arguments (max: 10 args prior to protocol 2.5)
                            // return param1 is return value
  PTP_CHDK_TempData,        // data is data to be stored for later
                            // param2 is for the TD flags below
  PTP_CHDK_UploadFile,      // data is 4-byte length of filename, followed by filename and contents
  PTP_CHDK_DownloadFile,    // preceded by PTP_CHDK_TempData with filename
                            // return data are file contents
  PTP_CHDK_ExecuteScript,   // data is script to be executed
                            // param2 is language of script
                            //  in proto 2.6 and later, language is the lower byte, rest is used for PTP_CHDK_SCRIPT_FL* flags
                            // return param1 is script id, like a process id
                            // return param2 is status from ptp_chdk_script_error_type
  PTP_CHDK_ScriptStatus,    // Script execution status
                            // return param1 bits
                            // PTP_CHDK_SCRIPT_STATUS_RUN is set if a script running, cleared if not
                            // PTP_CHDK_SCRIPT_STATUS_MSG is set if script messages from script waiting to be read
                            // all other bits and params are reserved for future use
  PTP_CHDK_ScriptSupport,   // Which scripting interfaces are supported in this build
                            // param1 CHDK_PTP_SUPPORT_LUA is set if lua is supported, cleared if not
                            // all other bits and params are reserved for future use
  PTP_CHDK_ReadScriptMsg,   // read next message from camera script system
                            // return param1 is chdk_ptp_s_msg_type
                            // return param2 is message subtype:
                            //   for script return and users this is ptp_chdk_script_data_type
                            //   for error ptp_chdk_script_error_type
                            // return param3 is script id of script that generated the message
                            // return param4 is length of the message data.
                            // return data is message.
                            // A minimum of 1 bytes of zeros is returned if the message has no data (empty string or type NONE)
  PTP_CHDK_WriteScriptMsg,  // write a message for scripts running on camera
                            // input param2 is target script id, 0=don't care. Messages for a non-running script will be discarded
                            // data length is handled by ptp data phase
                            // input messages do not have type or subtype, they are always a string destined for the script (similar to USER/string)
                            // output param1 is ptp_chdk_script_msg_status
  PTP_CHDK_GetDisplayData,  // Return camera display data
                            // This is defined as separate sub protocol in live_view.h
                            // Changes to the sub-protocol will always be considered a minor change to the main protocol
                            //  param2 bitmask of data
                            //  output param1 = total size of data
                            //  return data is protocol information, frame buffer descriptions and selected display data
                            //  Currently a data phase is always returned. Future versions may define other behavior
                            //  for values in currently unused parameters.
  // Direct image capture over USB.
  // Use lua get_usb_capture_support for available data types, lua init_usb_capture for setup
  PTP_CHDK_RemoteCaptureIsReady, // Check if data is available
                                 // return param1 is status
                                 //  0 = not ready
                                 //  0x10000000 = remote capture not initialized
                                 //  otherwise bitmask of PTP_CHDK_CAPTURE_* datatypes
                                 // return param2 is image number
  PTP_CHDK_RemoteCaptureGetData  // retrieve data
                                 // param2 is bit indicating data type to get
                                 // return param1 is length
                                 // return param2 more chunks available?
                                 //  0 = no more chunks of selected format
                                 // return param3 seek required to pos (-1 = no seek)
};
```

TODO: Document CHDK extensions
