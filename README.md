# CoDeSys_EIP
**What?**

CoDeSys_EIP is a CoDeSys 3.5.16.0 library that allows your CoDeSys controller (IPC) to communicate with various EtherNet/IP capable devices such as Allen Bradley / Rockwell programmable logic controller (PLC) through tag based communication or Fanuc robot with EIP set/get attributes; both via explicit mesaging.

**Why?**

In CoDeSys, the current method of communicating with the PLC is through implicit messaging.  This means you need to set up a generic EtherNet/IP (EIP) module on each end and for each task (input/output), where you specify the number of bytes for sending and receiving based on some form of polling (RPI) or triggered / event-based.  This is not very flexible as you will need to modify the PLC's code along with copying the address data into the EIP module buffer, and then repeat for the IPC... for each PLC that you want to connect to.  Similar for the Fanuc robot, there is no easy way to retrieve data from the robot to the Rockwell PLC unless the Enhanced Data Access (EDA) package is purchased; even then you are still bound to Rockwell.  This library allows your CoDeSys IPC to do (see `Examples\Fanuc\Fanuc_RPi.project`).

This library was first inspired by another library implemented in Python called [PyLogix](https://github.com/dmroeder/pylogix) and has been improved to handle STRUCTs and generic EIP services.  For the control engineers out there, you might already know that writing PLC code is not as flexible as writing higher level languages such as Python/Java/etc, where you can create variables with virtually any data type on the fly; thus, this library was heavily modified to fit into the controls realm.  It is written to be operate asynchronously (non-blocking) to avoid watchdog alerts, which means you make the call and get notified when data has been read/written succesfully.  If you need to read multiple variables quickly, you can create a lower priority task and place the calls into a WHILE loop to force operations in one scan cycle (see `Examples\Rockwell\Read-Write_Tags_RPi.project`).  At least 95% of the library leverages pointers for efficiency, so it might not be straight forward to digest at first.  The documentation / comments is not too bad, but feel free to raise issues if needed.

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
    
    _bWriteTag_codesys_bool_local   : BOOL; // program tag
    _stWriteTag_codesys_string      : stString; // STRUCT with string length (DINT) and string (82 is termination)
END_VAR
```

In your code, toggle `bEnable` of _PLC to `TRUE`.  There is optional `bAutoReconnect` if you want to reconnect in case session is terminated from idling.


#### Data alignment:
Allen Bradley is 4/8 bytes aligned, so make sure you specify CoDeSys STRUCTs with `{attribute 'pack_mode' := '4'}`.  Read the [CoDeSys pack mode](https://help.codesys.com/webapp/_cds_pragma_attribute_pack_mode;product=codesys;version=3.5.16.0).

#### Reading Tag
**NOTE**:
* Add "Program:{programName}." prefix to read program tags (e.g. Program:MainProgram.codesys_bool_local)
* Possible arguments for `bRead(psTag (POINTER TO STRING), eDataType (ENUM), pbBuffer (POINTER TO BYTE), uiSize (UINT), uiElements (UINT)), psId (POINTER TO STRING)`
    * psId is optional, but it is useful for troubleshooting. If you incorrectly declare your tag `codesys_boo` instead of `codesys_bool`, a `Path segment error` would typically be returned.  By declaring `psId:=ADR('Read#1: ')`, sError will return `'Read#1: Path segment error'` to let you know that something is wrong with the request
    * uiElements (default: `1`).  Will need to test if we can read multiple elements.
* If the data type of the read response does not match what the requested data type is, then a read error is thrown (avoids buffer overflow)
* `bRead` returns TRUE on successful read
* For those not familiar with ADR instruction, it retrieves the pointer location for you.  These example tags are hardcoded, but you can freely point to a STRING instead
    * **Example:** psTag:=ADR(_sMyTestString).  If string is empty, bRead returns `FALSE`

Below reads the PLC controller tag `codesys_bool` with data type of **BOOL** and writes to a CoDeSys **BOOL** called `_bReadTag_codesys_bool`
```
_PLC.bRead(psTag:=ADR('codesys_bool'),
            eDataType:=CoDeSys_EIP.eCipTypes._BOOL,
            pbBuffer:=ADR(_bReadTag_codesys_bool),
            psId:=ADR('Read#1: '));
```
Below reads the PLC controller tag `codesys_sint` with data type of **SINT** and writes to a CoDeSys **SINT** called `_siReadTag_codesys_sint`
```
_PLC.bRead(psTag:=ADR('codesys_sint'),
            eDataType:=CoDeSys_EIP.eCipTypes._SINT,
            pbBuffer:=ADR(_siReadTag_codesys_sint));
```
Below reads the PLC controller tag `codesys_usint` with data type of **USINT** and writes to a CoDeSys **USINT** called `_usiReadTag_codesys_usint`
```
_PLC.bRead(psTag:=ADR('codesys_usint'),
            eDataType:=CoDeSys_EIP.eCipTypes._USINT,
            pbBuffer:=ADR(_usiReadTag_codesys_usint));
```
Below reads the PLC controller tag `codesys_int` with data type of **INT** and writes to a CoDeSys **INT** called `_iReadTag_codesys_int`
```
_PLC.bRead(psTag:=ADR('codesys_int'),
            eDataType:=CoDeSys_EIP.eCipTypes._INT,
            pbBuffer:=ADR(_iReadTag_codesys_int));
```
Below reads the PLC controller tag `codesys_uint` with data type of **UINT** and writes to a CoDeSys **UINT** called `_uiReadTag_codesys_uint`
```
_PLC.bRead(psTag:=ADR('codesys_uint'),
            eDataType:=CoDeSys_EIP.eCipTypes._UINT,
            pbBuffer:=ADR(_uiReadTag_codesys_uint));
```
Below reads the PLC controller tag `codesys_dint` with data type of **DINT** and writes to a CoDeSys **DINT** called `_diReadTag_codesys_dint`
```
_PLC.bRead(psTag:=ADR('codesys_dint'),
            eDataType:=CoDeSys_EIP.eCipTypes._DINT,
            pbBuffer:=ADR(_diReadTag_codesys_dint));
```
Below reads the PLC controller tag `codesys_udint` with data type of **UDINT** and writes to a CoDeSys **UDINT** called `_udiReadTag_codesys_udint`
```
_PLC.bRead(psTag:=ADR('codesys_udint'),
            eDataType:=CoDeSys_EIP.eCipTypes._UDINT,
            pbBuffer:=ADR(_udiReadTag_codesys_udint));
```
Below reads the PLC controller tag `codesys_lint` with data type of **LINT** and writes to a CoDeSys **LINT** called `_liReadTag_codesys_lint`
```
_PLC.bRead(psTag:=ADR('codesys_lint'),
            eDataType:=CoDeSys_EIP.eCipTypes._LINT,
            pbBuffer:=ADR(_liReadTag_codesys_lint));
```
Below reads the PLC controller tag `codesys_ulint` with data type of **ULINT** and writes to a CoDeSys **ULINT** called `_uliReadTag_codesys_ulint`
```
_PLC.bRead(psTag:=ADR('codesys_ulint'),
            eDataType:=CoDeSys_EIP.eCipTypes._ULINT,
            pbBuffer:=ADR(_uliReadTag_codesys_ulint));
```
Below reads the PLC controller tag `codesys_real` with data type of **REAL** and writes to a CoDeSys **REAL** called `_rReadTag_codesys_real`
```
_PLC.bRead(psTag:=ADR('codesys_real'),
            eDataType:=CoDeSys_EIP.eCipTypes._REAL,
            pbBuffer:=ADR(_rReadTag_codesys_real));
```
Below reads the PLC controller tag `codesys_lreal` with data type of **LREAL** and writes to a CoDeSys **LREAL** called `_lrReadTag_codesys_lreal`
```
_PLC.bRead(psTag:=ADR('codesys_lreal'),
            eDataType:=CoDeSys_EIP.eCipTypes._LREAL,
            pbBuffer:=ADR(_lrReadTag_codesys_lreal));
```
Below reads the PLC controller tag `codesys_string` with data type of **STRUCT** and writes to a CoDeSys **STRING** called `_sReadTag_codesys_string`  
**Note**: If you do not specify `uiSize`, then the output will only contain the string data and not the string length (DINT)
```
_PLC.bRead(psTag:=ADR('codesys_string'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_sReadTag_codesys_string));
```
Below reads the PLC controller UDT `codesys_mixed` with data type of a **"complex" STRUCT** and writes to a CoDeSys **STRUCT** called `_stReadTag_codesys_mixed`  
**Note**: Specify the size of the CoDeSys STRUCT.  See `Examples` folder for more details
```
_PLC.bRead(psTag:=ADR('codesys_mixed'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stReadTag_codesys_mixed),
            uiSize:=SIZEOF(_stReadTag_codesys_mixed));
```
Below reads the PLC tag `codesys_string_local` of a program called `MainProgram` with data type of **STRUCT** and writes to a CoDeSys **STRUCT** called `_stReadTag_codesys_string_local`  
**Note**: Specify `uiSize` since STRUCT also has string length (DINT).  See `Examples` folder for more details
```
_PLC.bRead(psTag:=ADR('Program:MainProgram.codesys_string_local'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stReadTag_codesys_string_local),
            uiSize:=SIZEOF(_stReadTag_codesys_string_local));
```
Below reads the array index 3 of a PLC controller tag `testCaseFiveStrings` with data type of **STRING25** and writes to a CoDeSys **STRUCT** called `_stReadTag_testCaseFiveStrings`  
**Note**: Specify `uiSize` since STRUCT also has string length (DINT).  See `Examples` folder for more details
```
_PLC.bRead(psTag:=ADR('testCaseFiveStrings.strTest[3]'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stReadTag_testCaseFiveStrings),
            uiSize:=SIZEOF(_stReadTag_testCaseFiveStrings));
```

#### Writing Tag
**NOTE**:
* Writing all data types follows the same format as read.
* Add "Program:{programName}." prefix to write program tags (e.g. Program:MainProgram.codesys_bool_local)
* Possible arguments for `bWrite(psTag (POINTER TO STRING), eDataType (ENUM), pbBuffer (POINTER TO BYTE), uiSize (UINT), uiElements (UINT), psId (POINTER TO STRING))`
    * psId is optional, but it is useful for troubleshooting. If you incorrectly declare your tag `codesys_boo` instead of `codesys_bool`, a `Path segment error` would typically be returned.  By declaring `psId:=ADR('Write#1: ')`, sError will return `'Write#1: Path segment error'` to let you know that something is wrong with the request
    * uiElements (default: `1`).  Will need to test if we can write multiple elements.
* If you are writting to a tag that has not been read in yet, then an extra read request is performed first, and the STRUCT identifier of the response data is captured in `_stKnownStructs` "dictionary".  All subsequent writes of the same tag will perform a dictionary look up to save time.
* `bWrite` returns TRUE on successful write
* For those not familiar with ADR instruction, it retrieves the pointer location for you.  These example tags are hardcoded, but you can freely point to a STRING instead
    * **Example:** psTag:=ADR(_sMyTestString).  If string is empty, bWrite returns `FALSE`

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
**NOTE:** Writing to a PLC string must follow the format of a STRUCT made up of length (DINT) and a STRING.  Specify the length before writing!  See `Examples` folder for more details
```
_PLC.bWrite(psTag:=ADR('codesys_string'),
            eDataType:=CoDeSys_EIP.eCipTypes._STRUCT,
            pbBuffer:=ADR(_stWriteTag_codesys_string),
            uiSize:=SIZEOF(_stWriteTag_codesys_string));
```

#### Set/Get Attributes
**NOTE**:
* Similar to bRead/bWrite, you pass in argument and expect data to be written into a buffer
* You will need to create a STRUCT for each and specify what you are getting/setting
    * You could share the same STRUCT if space is a concern (i.e. use `_stAttributeList : CoDeSys_EIP.stAttributeList` for both set/get)
* Currently only `bGetAttributeSingle`, `bSetAttributeSingle`, `bGetAttributeList` and `bSetAttributeList` are implemented.  `bGetAttributeAll` and `bSetAttributeAll` will be next
* Possible arguments are `bFunctionType(stSTRUCT (STRUCT), pwAttributes (POINTER TO WORD), pbBuffer (POINTER TO BYTE), uiSize (UINT), pstId (POINTER TO STRING))`
    * `pwAttributes` is only available for `bGetAttributeList`
    * `pstId` is optional, but it is useful when troubleshooting and reading the _PLC.sError.  Available v1.0.1.0 and above
* Function will return TRUE on successful read/write
* See `Examples\Rockwell\Set-Get_Attribute_RPi.project` for more details
* **Examples:**
    * Get PLC audit value (checksum)
        * Create a STRUCT for get attribute single: `_stAttributeSingle	: CoDeSys_EIP.stAttributeSingle;`
        * Create a LWORD/ULINT for the result: `_lwAuditValue : LWORD;`
        * Specify parameters:
            * _stAttributeSingle.class := 16#8E; // 142
            * _stAttributeSingle.instance := 16#01; // 1
            * _stAttributeSingle.attribute := 16#1C; // 11
        * Call function `bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle, pbBuffer:=ADR(_lwAuditValue), uiSize:=SIZEOF(_lwAuditValue), pstId:=ADR('AuditValue: '))`
            * If there is an error, message will have header of `AuditValue`
    * Set PLC Change To Detect Mask
        * Create a STRUCT for get attribute single: `_stAttributeSingle	: CoDeSys_EIP.stAttributeSingle;`
        * Create a LWORD/ULINT for the value: `_lwMask : LWORD;`
        * Specify parameters:
            * _stAttributeSingle.class := 16#8E; // 142
            * _stAttributeSingle.instance := 16#01; // 1
            * _stAttributeSingle.attribute := 16#1C; // 11
        * Call function `bSetAttributeSingle(stAttributeSingle:=_stAttributeSingle, pbBuffer:=ADR(_lwMask), uiSize:=SIZEOF(_lwMask), pstId:=ADR('Mask: '))`
            * If there is an error, message will have header of `Mask`
    * Get the PLC time (look at `bGetPlcTime()` to see how it is implemented)
        * Create a STRUCT for get attribute list: `_stAttributeList : CoDeSys_EIP.stAttributeList`
        * Create an array of UINT/WORD or STRUCT for attributes to fetch: `_auiAttributes : ARRAY [1..5] OF UINT;`
        * Create a STRUCT to store result: `_stPlcTime : stPlcTime;`
        * Specify parameters:
            * _stAttributeList.class := 16#8B; // 139
            * _stAttributeList.instance := 16#01; // 1
            * _stAttributeList.attributeCount := 16#01; // 1
            * _auiAttributes[1] := 16#0B; // we are only getting one attribute here
        * Call function `bGetAttributeList(stAttributeList:=_stAttributeList, pwAttributes:=ADR(_auiAttributes), pbBuffer:=ADR(_stPlcTime), uiSize:=SIZEOF(_stPlcTime), pstId:=ADR('GetPlcTime: '))`
    * Set the PLC time (look at `bSetPlcTime(ULINT)` to see how it is implemented)
        * Create a STRUCT for get attribute list: `stAttributeList : CoDeSys_EIP.stAttributeList`
        * Create a STRUCT that has attribute (UINT) and time (ULINT): `_stSetPlcTime : stSetPlcTime;`
        * Specify parameters:
            * stAttributeList.class := 16#8B; // 139
            * stAttributeList.instance := 16#01; // 1
            * stAttributeList.attributeCount := 16#01; // 1
            * stAttributeList.attributeType := 16#06; // 6
        * Call function `bSetAttributeList(stAttributeList:=stAttributeList, pbBuffer:=ADR(_uliMyTime), uiSize:=SIZEOF(_uliMyTime), pstId:=ADR('SetPlcTime: '))`

### But does it work?
Yes... 60% of the time, it works every time.  Testing was done using a Raspberry Pi 3, with CoDeSys 3.5.16.0 runtime installed, to communicate with a Rockwell `5069-L330ERMS2/A` safety PLC over WiFi.  Each read/write instruction averaged approximately 3.7 milliseconds in test project (see `Examples\Rockwell\Read-Write_Tags_RPi.project`), but your mileage might vary.  You will need to install SysTime and OSCAT Basic (this is for time formatting).

### Current features (might add more, so put in your request)

#### List Identity
`bGetListIdentity()` (BOOL) is automatically called after TCP connection is established to return device info. You could scan your network for other EtherNet/IP capable devices.  
* **Examples:**
    * Retrieve single parameter using: `_sVendorId := _PLC.sVendorId` as STRING
    * Retrieve single parameter using: `_uiVendorId := _PLC.uiVendorId` as UINT
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
	* Retrieve time as STRING: `_sPlcTime := _PLC.sPlcTime`
		* **Output:** 'LDT#2020-07-10-01:05:59.409036000'
    * Retrieve time in microseconds as ULINT: `_uliPlcTime := _PLC.uliPlcTime`
    	* **Output:** 1593815478238754

`bSetPlcTime(ULINT)` (BOOL) sets the PLC time.
* **Examples:**
    * Synchronize PLC's time to IPC's time: `bSetPlcTime()`
    * Set a PLC time in microseconds to `Friday, July 3, 2020 10:31:18 PM GMT` with seconds level accuracy: `bSetPlcTime(1593815478000000)`
    * **NOTE:** Look at built-in `TimeStamp` function block

#### Detect Code Changes
From a security perspective, it is useful to detect changes on the Rockwell PLC.
`bGetPlcAuditValue()` (BOOL) requests the PLC audit value.
* **Example:**
	* Retrieve audit value as ULINT: `_uliAuditValue := _PLC.uliAuditValue`
		* **Output:** 12650121977826373092 (16#AF8E42EA65B3EDE4)
	* Retrieve audit value as STRING: `_sAuditValue := _PLC.sAuditValue`
		* **Output:** '0xAF8E42EA65B3EDE4'

`bGetPlcMask()` (BOOL) requests the PLC Change To Detect mask
* **Example:**
	* Retrieve mask value as ULINT: `_uliMask := _PLC.uliMask`
		* **Output:** 18446744073709551615 (16#FFFFFFFFFFFFFFFF)
	* Retrieve mask value as STRING: `_sMask := _PLC.sMask`
		* **Output:** '0xFFFFFFFFFFFFFFFF'

`bSetPlcMask(ULINT)` (BOOL) *should* set the PLC Change To Detect mask
**NOTE:** currently throwing a `Privelege Violation` error, need to investigate
* **Examples:**
    * Set the mask to 0xFFFF: `bSetPlcTime(65535)`

### Useful parameters (SET/GET)
**NOTE:** There are a lot more, so dive into library to see what works best for you
* `bAutoReconnect` (BOOL) reconnects you if the session closes after idle (no read/write request) for roughly 60 seconds
	* Default: `FALSE`
    * Example set: `_PLC.bAutoReconnect := TRUE;`
    * Example get: `_bReconnect := _PLC.bAutoReconnect;`
* `bHexPrefix` (BOOL) attaches the '0x' prefix to the strings from list identity STRUCT
	* Default: `TRUE`
* `bOverrideCheck` (BOOL) disables strict device type checking
	* Default: `FALSE`
    * Example: if the device is a barcode scanner that does not handle CIP, then default is not to continue with Register Session and Forward Open request and stop after list identity service.
    * Enable this if your device is not a Rockwell PLC but can communicate using explicit messaging (e.g. Fanuc robot).
* `bMicro800` (BOOL) specifies the device as a Micro800
	* Default: `FALSE`
    * Not tested.  Need someone with an actual Micro800 PLC
* `bProcessorSlot` (BYTE) specifies the processor slot
	* Default: `0`
* `sDeviceIp` (STRING) allows you to change device IP from the one specified initially
	* Default: `11.200.0.10`
    * Might allow you to use one function block instance to connect to multiple PLCs if speed is not a requirement but space is.
* `uiDevicePort` (UINT) allows you to change device port from the one specified initially
	* Default: `44818`
* `udiTcpWriteTimeout` (UDINT) specifies the maximum time it should take for the TCP client write to finish
	* Default: `200000 microseconds`
* `uiCipWriteTimeout` (UINT) specifies the maximum time it should take for a CIP write to finish
	* Default: `200 milliseconds`
* `udiTcpClientTimeOut` (UDINT) specifies the TCP client timeout
	* Default: `500000 microseconds`
* `udiTcpClientRetry` (UDINT) specifies the auto reconnect interval for `bAutoReconnect`
	* Default: `5000 milliseconds`

### Useful functions:
* `bCloseSession()` (BOOL) sends forward close request and then unregister session request. 
    * **NOTE:** If `bAutoReconnect` is set to TRUE, session will be re-established within the specified `udiTcpClientRetry` period
* `bResetFault()` (BOOL) call this to ACK read/write fault flags
* `bBuildRoute(STRING)` (BOOL) if you have a custom route.
    * Example: argument of ENBT1 -> ENBT2 -> PLC might look like `'4,192.168.1.219'`

**PS**: In memories of ztoka (April 2012 - May 2020).  This is for you bud.