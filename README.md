# CoDeSys_EIP
**What?**

CoDeSys_EIP is a CoDeSys 3.5.16.0 library that allows your CoDeSys controller (IPC) to communicate with various EtherNet/IP capable devices such as Allen Bradley / Rockwell programmable logic controller (PLC) through tag based communication or Fanuc robot with EIP set/get attributes; both via explicit messaging.

**Why?**

In CoDeSys, the current method of communicating with the PLC is through implicit messaging.  This means you need to set up a generic EtherNet/IP (EIP) module on each end and for each task (input/output), where you specify the number of bytes for sending and receiving based on some form of polling (RPI) or triggered / event-based.  This is not very flexible as you will need to modify the PLC's code along with copying the address data into the EIP module buffer, and then repeat for the IPC... for each PLC that you want to connect to.  Similar for the Fanuc robot, there is no easy way to retrieve data from the robot to the Rockwell PLC unless the Enhanced Data Access (EDA) package is purchased; even then you are still bound to Rockwell.  This library allows your CoDeSys IPC to do (see `Examples\Fanuc`).

This library was first inspired by another library implemented in Python called [PyLogix](https://github.com/dmroeder/pylogix) and has been improved to handle STRUCTs and generic EIP services.  For the control engineers out there, you might already know that writing PLC code is not as flexible as writing higher level languages such as Python/Java/etc, where you can create variables with virtually any data type on the fly; thus, this library was heavily modified to fit into the controls realm.  It is written to operate asynchronously (non-blocking) to avoid watchdog alerts, which means you make the call and be notified when data has been read/written succesfully.  If you need to read multiple variables quickly, you can create a lower priority task and place the calls into a WHILE loop to force operations in one scan cycle (see `Examples\Rockwell\Read-Write_Tags_RPi.project`).  At least 95% of the library leverages pointers for efficiency, so it might not be straight forward to digest at first.  The documentation / comments is not too bad, but feel free to raise issues if needed.

### Getting started
Create an function block instance in your CoDeSys program, and specify the PLC's IP and port.  Then create some variables:
```
VAR
    _PLC                            : CoDeSys_EIP.Device(sIpAddress:='192.168.1.219', uiPort:=44818);
    _bReadTag_codesys_bool          : BOOL;
    _siReadTag_codesys_sint         : SINT;
    _usiReadTag_codesys_usint       : USINT;
    _iReadTag_codesys_int           : INT;
    _uiReadTag_codesys_uint         : UINT;
    _diReadTag_codesys_dint         : DINT;
    _udiReadTag_codesys_udint       : UDINT;
    _liReadTag_codesys_lint         : LINT;
    _uliReadTag_codesys_ulint       : ULINT;
    _rReadTag_codesys_real          : REAL;
    _lrReadTag_codesys_lreal        : LREAL;
    _sReadTag_codesys_string        : STRING;
    _stReadTag_codesys_mixed        : stMixDatatype; // contains all data types above in random order
    _stReadTag_testCaseFiveStrings  : stString25; // custom string size of 25 chars
    _stReadTag_testCaseFiveStrings2 : ARRAY [1..2] OF stString25; // two stString25 elements for multi-read/write

    _bWriteTag_codesys_bool_local   : BOOL; // program tag
    _stWriteTag_codesys_string      : stString; // STRUCT with string length (DINT) and string (82 is termination)
END_VAR
```

In your code, toggle `bEnable` of _PLC to `TRUE`.  There is optional `bAutoReconnect` to re-establish session if terminated from idling.

#### Data alignment:
Allen Bradley is 4/8 bytes aligned, so make sure you specify CoDeSys STRUCTs with `{attribute 'pack_mode' := '4'}`.  Read the [CoDeSys pack mode](https://help.codesys.com/webapp/_cds_pragma_attribute_pack_mode;product=codesys;version=3.5.16.0).

#### Reading Tag
**NOTE**:
* Add "Program:{programName}." prefix to read program tags (e.g. Program:MainProgram.codesys_bool_local)
* Possible arguments for `bRead`:
    * `psTag` (POINTER TO STRING) [**required**]: Pointer to the string tag [e.g. psTag:=ADR('codesys_bool')]. If string is empty, bRead throws error and returns `FALSE`.
        * You can also point to a STRING instead [e.g. psTag:=ADR(_sMyTestString)].
    * `eDataType` (ENUM) [**required**]: Expected CIP data type.  If read response does not match, a read error is raised (avoids buffer overflow).
    * `pbBuffer` (POINTER TO BYTE) [**required**]: Pointer to the output buffer.
    * `uiSize` (UINT): Size of the output buffer (pbBuffer).
    * `uiElements` (UINT): Elements to be requested; used when specifying array.
        * Default: `1`
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Read#1: ')].
        * Useful for troubleshooting. If you incorrectly declare your tag `codesys_boo` instead of `codesys_bool`, a `Path Segment Error` would typically be returned.  By declaring `psId:=ADR('Read#1: ')`, sError will return `'Read#1: Path Segment Error'` to let you know of the CIP error.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* `bRead` returns TRUE on successful read.

Below reads the PLC controller tag `codesys_bool` with data type of **BOOL** and writes to a CoDeSys **BOOL** called `_bReadTag_codesys_bool`.
```
_PLC.bRead(psTag:=ADR('codesys_bool'),
            eDataType:=CoDeSys_EIP.eCipTypes._BOOL,
            pbBuffer:=ADR(_bReadTag_codesys_bool),
            psId:=ADR('Read#1: '));
```

*
*
*

Below reads the PLC controller tag `codesys_dint` with data type of **DINT** and writes to a CoDeSys **DINT** called `_diReadTag_codesys_dint`.
```
_PLC.bRead(psTag:=ADR('codesys_dint'),
            eDataType:=CoDeSys_EIP.eCipTypes._DINT,
            pbBuffer:=ADR(_diReadTag_codesys_dint));
```

*
*
*

Below reads the PLC controller tag `codesys_lreal` with data type of **LREAL** and writes to a CoDeSys **LREAL** called `_lrReadTag_codesys_lreal`.
```
_PLC.bRead(psTag:=ADR('codesys_lreal'),
            eDataType:=CoDeSys_EIP.eCipTypes._LREAL,
            pbBuffer:=ADR(_lrReadTag_codesys_lreal));
```
Below reads the PLC controller tag `codesys_string` with data type of **STRUCT** and writes to a CoDeSys **STRING** called `_sReadTag_codesys_string`.
**Note**: If you do not specify `uiSize`, then the output will only contain the string data and not the string length (DINT).
```
_PLC.bRead(psTag:=ADR('codesys_string'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_sReadTag_codesys_string));
```
Below reads the PLC controller UDT `codesys_mixed` with data type of a **"complex" STRUCT** and writes to a CoDeSys **STRUCT** called `_stReadTag_codesys_mixed`.
**Note**: Specify the size of the CoDeSys STRUCT.  See `Examples\Rockwell` folder for more details.
```
_PLC.bRead(psTag:=ADR('codesys_mixed'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stReadTag_codesys_mixed),
            uiSize:=SIZEOF(_stReadTag_codesys_mixed));
```
Below reads the PLC tag `codesys_string_local` of a program called `MainProgram` with data type of **STRUCT** and writes to a CoDeSys **STRUCT** called `_stReadTag_codesys_string_local`.
**Note**: Specify `uiSize` since STRUCT also has string length (DINT).  See `Examples\Rockwell` folder for more details.
```
_PLC.bRead(psTag:=ADR('Program:MainProgram.codesys_string_local'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stReadTag_codesys_string_local),
            uiSize:=SIZEOF(_stReadTag_codesys_string_local));
```
Below reads the array index 3 of a PLC controller tag `testCaseFiveStrings` with data type of **STRING25** and writes to a CoDeSys **STRUCT** called `_stReadTag_testCaseFiveStrings`.
**Note**: Specify `uiSize` since STRUCT also has string length (DINT).  See `Examples\Rockwell` folder for more details.
```
_PLC.bRead(psTag:=ADR('testCaseFiveStrings.strTest[3]'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stReadTag_testCaseFiveStrings),
            uiSize:=SIZEOF(_stReadTag_testCaseFiveStrings));
```
Below reads 2 elements (starting index of 1) from PLC controller tag `testCaseFiveStrings` with data type of **STRING25** and writes to a CoDeSys **STRUCT** called `_stReadTag_testCaseFiveStrings2`.
**Note**: Specify `uiSize` since STRUCT also has string length (DINT).  See `Examples\Rockwell` folder for more details.
```
_PLC.bRead(psTag:=ADR('testCaseFiveStrings.strTest[1]'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stReadTag_testCaseFiveStrings2),
            uiSize:=SIZEOF(_stReadTag_testCaseFiveStrings2),
            uiElements:=2);
```

#### Writing Tag
**NOTE**:
* Add "Program:{programName}." prefix to write program tags (e.g. Program:MainProgram.codesys_bool_local)
* If you are writting to a tag that has not been read in yet, then an extra read request is performed first, and the STRUCT identifier of the response data is captured in `_stKnownStructs` "dictionary".  All subsequent writes of the same tag will perform a dictionary look up to save time.
* Possible arguments for `bWrite`:
    * `psTag` (POINTER TO STRING) [**required**]: Pointer to the string tag [e.g. psTag:=ADR('codesys_bool')]. If string is empty, bWrite throws error and returns `FALSE`.
        * You can also point to a STRING instead [e.g. psTag:=ADR(_sMyTestString)].
    * `eDataType` (ENUM) [**required**]: Defined CIP data type.  If write response does not match, a write error is raised.
    * `pbBuffer` (POINTER TO BYTE) [**required**]: Pointer to the input buffer.
    * `uiSize` (UINT): Size of the input buffer (pbBuffer).
    * `uiElements` (UINT): Elements to be requested; used when specifying array.
        * Default: `1`
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Write#1: ')].
        * Useful for troubleshooting; output to _PLC.sError.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* `bWrite` returns TRUE on successful write

Below writes the CoDeSys **BOOL** called `_bWriteTag_codesys_bool_local` to the PLC tag `codesys_bool_local` of a program called `MainProgram`
```
_PLC.bWrite(psTag:=ADR('Program:MainProgram.codesys_bool_local'),
            eDataType:=CoDeSys_EIP.eCipTypes._BOOL,
            pbBuffer:=ADR(_bWriteTag_codesys_bool_local),
            psId:=ADR('Write#19: '));
```
*
*
*

Below writes the CoDeSys **LREAL** called `_lrWriteTag_codesys_lreal_local` to the PLC tag `codesys_lreal_local` of a program called `MainProgram`
```
_PLC.bWrite(psTag:=ADR('Program:MainProgram.codesys_lreal_local'),
            eDataType:=CoDeSys_EIP.eCipTypes._LREAL,
            pbBuffer:=ADR(_lrWriteTag_codesys_lreal_local));
```
Below writes the CoDeSys **STRUCT** called `_stWriteTag_codesys_string` to the PLC controller tag `codesys_string`.  
**NOTE:** Writing to a PLC string must follow the format of a STRUCT made up of length (DINT) and a STRING.  Specify the length before writing!  See `Examples\Rockwell` folder for more details
```
_PLC.bWrite(psTag:=ADR('codesys_string'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stWriteTag_codesys_string),
            uiSize:=SIZEOF(_stWriteTag_codesys_string));
```

#### Set/Get Attributes
**NOTE**:
* You will need to create a STRUCT and specify what you are getting/setting
    * You could share the same STRUCT if space is a concern (i.e. use `_stAttributeList : CoDeSys_EIP.stAttributeList` for both set/get)
* Possible arguments for `bGetAttributeAll`, `bSetAttributeAll`, `bGetAttributeSingle`, `bSetAttributeSingle`, `bGetAttributeList`, `bSetAttributeList`:
    * `stSTRUCT` (STRUCT) [**required**]:  Struct element.
    * `pbBuffer` (POINTER TO BYTE) [**required**]: Pointer to either input/output buffer based on type set/get.
    * `uiSize` (UINT) [**required**]: Size of input/output buffer (pbBuffer).
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('GetPlcTime: ')].
        * Useful for troubleshooting; output to _PLC.sError.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* Possible arguments for `bGenericService`: **typically populate either pbInBuffer/uiInsize or pbOutBuffer/uiOutSize**
    * `stSTRUCT` (STRUCT) [**required**]:  Struct element.
    * `pbInBuffer` (POINTER TO BYTE): Pointer to input buffer.
    * `uiInSize` (UINT): Size of input buffer.
    * `pbOutBuffer` (POINTER TO BYTE): Pointer to output buffer.
    * `uiOutSize` (UINT): Size of output buffer.
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('ExecPCCC: ')].
        * Useful for troubleshooting; output to _PLC.sError.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* Function returns TRUE on successful read/write
* See `Examples\Rockwell\Set-Get_Attribute_RPi.project` for more details
* **Examples:**
    * Create a re-usable STRUCT or one for each service type: `_stCipService : CoDeSys_EIP.stCipService;`
    * Get PLC audit value (checksum)
        * Create a LWORD/ULINT for the result: `_lwAuditValue : LWORD;`
        * Specify parameters:
            * _stCipService.class := 16#8E; // 142
            * _stCipService.instance := 16#01; // 1
            * _stCipService.attribute := 16#1C; // 28
        * Call function `bGetAttributeSingle(stCipService:=_stCipService, pbBuffer:=ADR(_lwAuditValue), uiSize:=SIZEOF(_lwAuditValue), pstId:=ADR('AuditValue: '))`
            * If there is an error, message will have header of `AuditValue`
    * Set PLC Change To Detect Mask
        * Create a LWORD/ULINT for the value: `_lwMask : LWORD;`
        * Specify parameters:
            * _stCipService.class := 16#8E; // 142
            * _stCipService.instance := 16#01; // 1
            * _stCipService.attribute := 16#1C; // 28
        * Call function `bSetAttributeSingle(stCipService:=_stCipService, pbBuffer:=ADR(_lwMask), uiSize:=SIZEOF(_lwMask), pstId:=ADR('Mask: '))`
            * If there is an error, message will have header of `Mask`
    * Get the PLC time (look at `bGetPlcTime()` to see how it is implemented)
        * Create a STRUCT to store result: `_stPlcTime : stPlcTime;`
        * Specify parameters:
            * _stCipService.class := 16#8B; // 139
            * _stCipService.instance := 16#01; // 1
            * _stCipService.attributeCount := 16#01; // 1
            * _stCipService.attributeList[1] := 16#0B; // we are only getting one attribute here
        * Call function `bGetAttributeList(stCipService:=_stCipService, pbBuffer:=ADR(_stPlcTime), uiSize:=SIZEOF(_stPlcTime), pstId:=ADR('GetPlcTime: '))`
    * Set the PLC time (look at `bSetPlcTime(ULINT)` to see how it is implemented)
        * Create a ULINT that stores time in microseconds: `_uliSetTime : ULINT;`
        * Specify parameters:
            * _stCipService.class := 16#8B; // 139
            * _stCipService.instance := 16#01; // 1
            * _stCipService.attributeCount := 16#01; // 1
            * _stCipService.attributeList[1] := 16#06; // 6
        * Call function `bSetAttributeList(stCipService:=_stCipService, pbBuffer:=ADR(_uliSetTime), uiSize:=SIZEOF(_uliSetTime), pstId:=ADR('SetPlcTime: '))`

### But does it work?
Yes... 60% of the time, it works every time.  Testing was done using a Raspberry Pi 3, with CoDeSys 3.5.16.0 runtime installed, to communicate with a Rockwell `5069-L330ERMS2/A` safety PLC over WiFi.  Each read/write instruction averaged approximately 3.7 milliseconds in test project (see `Examples\Rockwell\Read-Write_Tags_RPi.project`), but your mileage might vary.  You will need to install SysTime and OSCAT Basic (this is for time formatting).

### Current features

#### List Identity
`bGetListIdentity()` (BOOL) is automatically called after TCP connection is established to return device info. You could scan your network for other EtherNet/IP capable devices.  
* **Examples:**
    * Retrieve single parameter as UINT using: `_uiVendorId := _PLC.uiVendorId`
        * **Output:** `1`
    * Retrieve single parameter as STRING using: `_sVendorId := _PLC.sVendorId`
        * **Output:** `'Rockwell Automation/Allen-Bradley'`
    * Retrieve entire STRUCT using: `_stDevice := _PLC.stListIdentity`
        * Requires STRUCT variable: `_stDevice`: `CoDeSys_EIP.stListIdentity`
        * **Output:**
            * encapsulationVersion: `1`
            * vendorId: `'Rockwell Automation/Allen-Bradley'`
            * deviceType: `'Programmable Logic Controller'`
            * productCode: `223`
            * revision: `'32.11'`
            * status: `'0x3060'`
            * serialNumber: `'0x60D789F4'`
            * productName: `'5069-L330ERMS2/A'`
            * state: `3`

#### Get/Set PLC time
`bGetPlcTime()` (BOOL) requests the current PLC time.  The function can handle 64b time up to nanoseconds, but the PLC's accuracy is only available at the microseconds.
* **Example:**
    * Retrieve time as STRING: `_sPlcTime := _PLC.sPlcTime`.
        * **Output:** `'LDT#2020-07-10-01:05:59.409036000'`
    * Retrieve time in microseconds as ULINT: `_uliPlcTime := _PLC.uliPlcTime`.
        * **Output:** `1593815478238754`

`bSetPlcTime(ULINT)` (BOOL) sets the PLC time.
* **Examples:**
    * Synchronize PLC's time to IPC's time: `bSetPlcTime()`.
    * Set a PLC time in microseconds to `Friday, July 3, 2020 10:31:18 PM GMT` with seconds level accuracy: `bSetPlcTime(1593815478000000)`.
    * **NOTE:** Look at built-in `Timestamp` function block.

#### Detect Code Changes
From a security perspective, it is useful to detect changes on the Rockwell PLC.
`bGetPlcAuditValue()` (BOOL) requests the PLC audit value.
* **Example:**
    * Retrieve audit value as ULINT: `_uliAuditValue := _PLC.uliAuditValue`.
        * **Output:** `12650121977826373092` (16#AF8E42EA65B3EDE4)
    * Retrieve audit value as STRING: `_sAuditValue := _PLC.sAuditValue`.
        * **Output:** `'0xAF8E42EA65B3EDE4'`

`bGetPlcMask()` (BOOL) requests the PLC Change To Detect mask.
* **Example:**
    * Retrieve mask value as ULINT: `_uliMask := _PLC.uliMask`.
        * **Output:** `18446744073709551615` (16#FFFFFFFFFFFFFFFF)
    * Retrieve mask value as STRING: `_sMask := _PLC.sMask`.
        * **Output:** `'0xFFFFFFFFFFFFFFFF'`

`bSetPlcMask(ULINT)` (BOOL) *should* set the PLC Change To Detect mask.

**NOTE:** currently throwing a `Privilege Violation` error, need to investigate.
* **Examples:**
    * Set the mask to 0xFFFF: `bSetPlcMask(65535)` or `bSetPlcMask(16#FFFF)`

### Useful parameters (SET/GET)
**NOTE:** There are a lot more, so dive into library to see what works best for you
* `bAutoReconnect` (BOOL) reconnects you if the session closes after idle (no read/write request) for roughly 60 seconds.
    * Default: `FALSE`
    * Example set: `_PLC.bAutoReconnect := TRUE;`
    * Example get: `_bReconnect := _PLC.bAutoReconnect;`
* `bHexPrefix` (BOOL) attaches the '0x' prefix to the strings from list identity STRUCT.
    * Default: `TRUE`
* `bUnconnectedMessaging` (BOOL) skips forward open and default to use unconnected messaging (Send RR Data).
    * Default: `FALSE`
* `bMicro800` (BOOL) specifies the device as a Micro800.
    * Default: `FALSE`
    * Not tested.  Need someone with an actual Micro800 PLC.
* `bProcessorSlot` (BYTE) specifies the processor slot.
    * Default: `0`
* `sDeviceIp` (STRING) allows you to change device IP from the one specified initially.
    * Default: `11.200.0.10`
    * Toggle bEnable to update settings
* `uiDevicePort` (UINT) allows you to change device port from the one specified initially.
    * Default: `44818`
    * Toggle bEnable to update settings
* `udiTcpWriteTimeout` (UDINT) specifies the maximum time it should take for the TCP client write to finish.
    * Default: `200000` microseconds
* `uiCipRequestTimeout` (UINT) specifies the maximum time it should take for a CIP request to finish.
    * Default: `200` milliseconds
* `udiTcpClientTimeOut` (UDINT) specifies the TCP client timeout.
    * Default: `500000` microseconds
* `udiTcpClientRetry` (UDINT) specifies the auto reconnect interval for `bAutoReconnect`.
    * Default: `5000` milliseconds
* `uiConnectionSize` (UINT) specifies the maximum connection bytes size for each read/write transaction.  Value is automatically resized to gvcParameters.uiTcpBuffer if it exceeds it.
    * Default: `508`
        * Using 508, which is divisible by 4.
    * **NOTE:** Large forward open has value greater than 511, and values of over 4000 seems to return `Resource Unavailable` error on CompactLogix.
* `stRoute` (STRUCT) specifies a non-standard route that cannot be generated by `bBuildRoute(STRING)`.
    * **NOTE:** Max route length is 96 bytes, and make sure to specify STRUCT length so forward open knows how to properly build the route.

### Useful functions:
* `bCloseSession()` (BOOL) sends forward close request and then unregister session request. 
    * **NOTE:** If `bAutoReconnect` is set to TRUE, session will be re-established within the specified `udiTcpClientRetry` period.
* `bResetFault()` (BOOL) call this to ACK read/write fault flags.
* `bBuildRoute(STRING)` (BOOL) if you have a custom route.
    * Example: argument of ENBT1 -> ENBT2 -> PLC might look like `'4,192.168.1.219'`.

**PS**: In memories of ztoka (April 2012 - May 2020).  This is for you bud.