# CHDK PTP Extension
[CHDK](https://chdk.fandom.com/wiki/CHDK) implements a custom opcode `0x9999` with a generic low-level interface to communicate with the camera's OS.
The primary client implementation for this opcode is in the [chdkptp](https://app.assembla.com/wiki/show/chdkptp) project.

- Responder-side implementation: https://github.com/petabyt/chdk/blob/23c1f3f04fa1c7d483083d403818972a739c6dfd/core/ptp.c#L443 
- All functionality is switched from a code in the first parameter: https://github.com/petabyt/chdk/blob/master/trunk/core/ptp.c#L33C1-L33C1 
- [ptp.c](https://github.com/petabyt/chdk/blob/master/trunk/core/ptp.c)
- [ptp.h](https://github.com/petabyt/chdk/blob/master/trunk/core/ptp.h)

## `PTP_CHDK_Version` - `0`
- **Return Parameters**:
  - **param1**: Major version number.
  - **param2**: Minor version number.

## `PTP_CHDK_GetMemory` - `1`
- **Input Parameters**:
  - **param2**: Base address (direct access may fail for MMIO or special memory locations; use buffered mode in these cases).
  - **param3**: Size in bytes of the memory block to retrieve.
  - **param4**: Options for the operation:
    - `0`: Read memory directly.
    - `1`: Read memory using a buffered operation.
    - Other values are reserved for future use.
- **Return Data**:
  - Memory block corresponding to the specified address and size.

## `PTP_CHDK_SetMemory` - `2`
- **Input Parameters**:
  - **param2**: Address of the memory block to set.
  - **param3**: Size in bytes of the new memory block.
  - **Data**: New memory block content.

## `PTP_CHDK_CallFunction` - `3`
- Calls a function from a 32-bit pointer in memory, with 32-bit arguments.
- **Input Data**:
  - Array of function pointer and up to 10 32-bit integer arguments.
- **Return Parameters**:
  - **param1**: Return value of the function.

## `PTP_CHDK_TempData` - `4`
- Can be used to store or download temporary data in a temporary location the camera's RAM 
- **Input Data**:
  - Arbitrary data to be stored for later use.
- **Input Parameters**:
  - **param2**: Flags for storing data.

## `PTP_CHDK_UploadFile` - `5`
- **Input Data**:
  - 4-byte length of the filename, followed by the filename and file contents. Writes the rest of payload to disk.

## `PTP_CHDK_DownloadFile` - `6`
- **Input**:
  - Preceded by `PTP_CHDK_TempData` with the filename.
  - Call `PTP_CHDK_TempData` with a string for the filename.
- **Return Data**:
  - File contents.

## `PTP_CHDK_ExecuteScript` - `7`
- **Input Data**:
  - Script content to be executed.
- **Input Parameters**:
  - **param2**: Script language.
    - For protocol 2.6 and later, the lower byte specifies the language, while the rest is used for `PTP_CHDK_SCRIPT_FL*` flags.
- **Return Parameters**:
  - **param1**: Script ID (similar to a process ID).
  - **param2**: Status, as defined in `ptp_chdk_script_error_type`.

## `PTP_CHDK_ScriptStatus` - `8`
- **Return Parameters**:
  - **param1**: Bitmask of script execution status:
    - `PTP_CHDK_SCRIPT_STATUS_RUN`: Set if a script is running.
    - `PTP_CHDK_SCRIPT_STATUS_MSG`: Set if messages are waiting to be read from the script.
  - Other bits and parameters are reserved for future use.

## `PTP_CHDK_ScriptSupport` - `9`
- **Return Parameters**:
  - **param1**: Bitmask of supported scripting interfaces:
    - `CHDK_PTP_SUPPORT_LUA`: Set if Lua is supported.

## `PTP_CHDK_ReadScriptMsg` - `10`
- **Return Parameters**:
  - **param1**: `chdk_ptp_s_msg_type`.
  - **param2**: Message subtype:
    - For script returns and user messages, this is `ptp_chdk_script_data_type`.
    - For errors, this is `ptp_chdk_script_error_type`.
  - **param3**: Script ID of the script that generated the message.
  - **param4**: Length of the message data.
- **Return Data**:
  - Message content. If the message has no data, a minimum of 1 byte of zeros is returned.

## `PTP_CHDK_WriteScriptMsg` - `11`
- **Input Parameters**:
  - **param2**: Target script ID (`0` for "don't care"; messages for non-running scripts are discarded).
- **Input Data**:
  - String message for the script.
- **Output Parameters**:
  - **param1**: `ptp_chdk_script_msg_status`.

## `PTP_CHDK_GetDisplayData` - `12`
- **Input Parameters**:
  - **param2**: Bitmask of requested data.
- **Output Parameters**:
  - **param1**: Total size of data.
- **Return Data**:
  - Protocol information, frame buffer descriptions, and selected display data.

## `PTP_CHDK_RemoteCaptureIsReady` - `13`
- **Return Parameters**:
  - **param1**: Status:
    - `0`: Not ready.
    - `0x10000000`: Remote capture not initialized.
    - Otherwise, bitmask of `PTP_CHDK_CAPTURE_*` data types.
  - **param2**: Image number.

## `PTP_CHDK_RemoteCaptureGetData` - `14`
- **Input Parameters**:
  - **param2**: Bit indicating data type to retrieve.
- **Return Parameters**:
  - **param1**: Length of the data.
  - **param2**: Indicates if more chunks are available (`0` = no more chunks).
  - **param3**: Seek position (`-1` = no seek required).


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
