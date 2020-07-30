# CoDeSys_EIP - Fanuc
If you need to handle more than just discrete inputs/outputs, CoDeSys_EIP allows you to read/write robot variables such as numeric registers, position registers, and string registers.

### Getting started
Create an function block instance in your CoDeSys program, and specify the robot's IP and port.  Then create some variables:
```
VAR
    _Robot              : CoDeSys_EIP.Device(sIpAddress:='192.168.1.220', uiPort:=44818);
    _uiIndex            : UINT;                 // state machine
    _bReadFinished      : BOOL;                 // read status state
    _bWriteFinished     : BOOL;                 // write status state
    _bGoodRead          : BOOL;                 // status of read tag
    _bGoodWrite         : BOOL;                 // status of write tag

    _stCipService       : CoDeSys_EIP.stCipService;
    _stLPOS             : stCartesianPosition;  // current linear position
    _stJPOS             : stJointPosition;      // current joint position
    _stPosReg1          : stCartesianPosition;  // position register
    _stPosReg2          : stJointPosition;      // position register

    _rNumReg            : REAL;                 // test numeric register
    _diNumReg           : DINT;                 // test numeric register
    _stStringRegister   : stStringRegister;     // test string register
    
    _stAllRegsInt       : stAllRegistersINT;    // contains 124 registers
    _stBlockRegsInt     : stBlockRegistersINT;  // contains 10 registers
END_VAR
```

In your code, toggle `bEnable` of _Robot to `TRUE`.  There is optional `bAutoReconnect` to re-establish session if terminated from idling (looks like robot controller never closes session).  You need to specify Unconnected Messaging (Send RR Data) only via `_Robot.bUnconnectedMessaging := TRUE` (see `Fanuc_RPi.project`)!

#### Data alignment:
Fanuc robot follows similar layout like Rockwell with 4 bytes alignment when we have gaps (not 4B divisible), so make sure you specify CoDeSys STRUCTs with `{attribute 'pack_mode' := '4'}` when appropriate.  Read the [CoDeSys pack mode](https://help.codesys.com/webapp/_cds_pragma_attribute_pack_mode;product=codesys;version=3.5.16.0).

#### Available services
Below is a table that describes possible operations

```
=====================
Service Code:
=====================
* 0x01 (1)      : get attribute all
* 0x02 (2)      : set attribute all
* 0x0E (14)     : get attribute single
* 0x10 (16)     : set attribute single
* 0x32 (50)     : get attribute block
* 0x33 (51)     : set attribute block
=====================
Class Code:
=====================
* 0x6B (107)    : numeric register (integer)
* 0x6C (108)    : numeric register (real)
* 0x6D (109)    : string register
* 0x7B (123)    : position register (Cartesian)
* 0x7C (124)    : position register (joint)
* 0x7D (125)    : current position (Cartesian)
* 0x7E (126)    : current position (joint)
=====================
Instance:
=====================
* 0x01 (1)      : single item
* 0x(len)(grp)  : block items (e.g. 0x0501 (1281) is for writing 5 items of group 1)
=====================
Attribute:
=====================
* 0x01 (1)      : index of single item or starting index of block items
```

#### Reading Values
**NOTE**:
* Possible arguments `bGetAttributeAll`, `bGetAttributeSingle`:
    * `stSTRUCT` (STRUCT) [**required**]:  Struct element.
    * `pbBuffer` (POINTER TO BYTE): Pointer to output buffer.
    * `uiSize` (UINT): Size of output buffer.
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Read#1: ')].
        * Useful for troubleshooting; output to _PLC.sError.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* Possible arguments `bGenericService`: **Used for get attribute block**
    * `stSTRUCT` (STRUCT) [**required**]:  Struct element.
    * `pbOutBuffer` (POINTER TO BYTE): Pointer to output buffer.
    * `uiOutSize` (UINT): Size of output buffer.
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Read#1: ')].
        * Useful for troubleshooting; output to _PLC.sError.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* Function returns TRUE on successful read.
* Numeric value will be casted based on your request.
    * **Example:** Numeric register #5 has value of `11.111`.  If you request integer (class := 16#6B), then return value will be `11`.  If you request real (class := 16#6C), then return value will be `11.111`.
* String register holds up to 253 characters.
* **Position registers**:
    * UserFrame has value of 0x3F (63) and ToolFrame of 0x1F (31).
    * Reading:
        * If you read an uninitialized register, X/Y/Z/W/P/R/EXT1/EXT2/EXT3 (Cartesian) or J1/J2/J3/J4/J5/J6/J7/J8/J9 (joint) will return not a number (NaN).

Below reads numeric register index 5, which is a **DINT** and writes to a CoDeSys **DINT** called `_diNumReg`
```
_stCipService.class := 16#6B; // numeric register (integer)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#05; // register index 5
_bGoodRead := _Robot.bGetAttributeSingle(stCipService:=_stCipService,
                                        pbBuffer:=ADR(_diNumReg),
                                        uiSize:=SIZEOF(_diNumReg),
                                        psId:=ADR('Read R#5: '));
```
Below reads numeric register index 10, which is a **REAL** and writes to a CoDeSys **REAL** called `_rNumReg`
```
_stCipService.class := 16#6C; // numeric register (real)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#0A; // register index 10
_bGoodRead := _Robot.bGetAttributeSingle(stCipService:=_stCipService,
                                        pbBuffer:=ADR(_rNumReg),
                                        uiSize:=SIZEOF(_rNumReg),
                                        psId:=ADR('Read R#10: '));
```
Below reads string register index 5, which is a **STRUCT** and writes to a CoDeSys **STRUCT** called `_stStringRegister`
```
_stCipService.class := 16#6D; // string register
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#05; // register index 5
_bGoodRead := _Robot.bGetAttributeSingle(stCipService:=_stCipService,
                                        pbBuffer:=ADR(_stStringRegister),
                                        uiSize:=SIZEOF(_stStringRegister),
                                        psId:=ADR('Read SR#5: '));
```
Below reads position register index 1, which is a **CARTESIAN** position and writes to a CoDeSys **STRUCT** called `_stPosReg1`
```
_stCipService.class := 16#7B; // position register (Cartesian)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#01; // register index 1
_bGoodRead := _Robot.bGetAttributeSingle(stCipService:=_stCipService,
                                        pbBuffer:=ADR(_stPosReg1),
                                        uiSize:=SIZEOF(_stPosReg1),
                                        psId:=ADR('Read PR#1: '));
```
Below reads position register index 2, which is a **JOINT** position and writes to a CoDeSys **STRUCT** called `_stPosReg2`
```
_stCipService.class := 16#7C; // position register (joint)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#02; // register index 2
_bGoodRead := _Robot.bGetAttributeSingle(stCipService:=_stCipService,
                                        pbBuffer:=ADR(_stPosReg2),
                                        uiSize:=SIZEOF(_stPosReg2),
                                        psId:=ADR('Read PR#2: '));
```
Below reads the robot's position in **CARTESIAN** format and writes to a CoDeSys **STRUCT** called `_stLPOS`
```
_stCipService.class := 16#7D; // current position (Cartesian)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#01; // must be 1
_bGoodRead := _Robot.bGetAttributeSingle(stCipService:=_stCipService,
                                        pbBuffer:=ADR(_stLPOS),
                                        uiSize:=SIZEOF(_stLPOS),
                                        psId:=ADR('Read LPOS: '));
```
Below reads the robot's position in **JOINT** format and writes to a CoDeSys **STRUCT** called `_stJPOS`
```
_stCipService.class := 16#7E; // current position (joint)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#01; // must be 1
_bGoodRead := _Robot.bGetAttributeSingle(stCipService:=_stCipService,
                                        pbBuffer:=ADR(_stJPOS),
                                        uiSize:=SIZEOF(_stJPOS),
                                        psId:=ADR('Read JPOS: '));
```
Below reads a block of 10 registers, starting index of 1, as integer and writes to a CoDeSys **STRUCT** called `_stBlockRegsInt`
```
_stCipService.cipService := 16#32; // get attribute block
_stCipService.class := 16#6B; // numeric register (integer)
_stCipService.instance := 16#0A01; // 10 (16#0A) elements of group 1 (16#01)
_stCipService.attribute := 16#01; // starting index 1
_bGoodRead := _Robot.bGenericService(stCipService:=_stCipService,
                                    pbOutBuffer:=ADR(_stBlockRegsInt),
                                    uiOutSize:=SIZEOF(_stBlockRegsInt),
                                    psId:=ADR('Read 10 reg: '));
```

#### Writing Values
**NOTE**:
* Possible arguments `bSetAttributeAll`, `bSetAttributeSingle`:
    * `stAttribute` (STRUCT) [**required**]: Struct element.
    * `pbBuffer` (POINTER TO BYTE): Pointer to input buffer.
    * `uiSize` (UINT): Size of input buffer.
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Write#1: ')].
        * Useful for troubleshooting; output to _Robot.sError.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* Possible arguments `bGenericService`: **Used for set attribute block**
    * `stSTRUCT` (STRUCT) [**required**]:  Struct element.
    * `pbInBuffer` (POINTER TO BYTE): Pointer to input buffer.
    * `uiInSize` (UINT): Size of input buffer.
    * `psId` (POINTER TO STRING): Pointer to caller id [e.g. psId:=ADR('Write#1: ')].
        * Useful for troubleshooting; output to _PLC.sError.
    * `bUnconnected` (BOOL): Forces unconnected messaging (Send RR Data) if `TRUE`.
* Function returns TRUE on successful write
* String register holds up to 253 characters.
* **Position registers**:
    * UserFrame has value of 0x3F (63) and ToolFrame of 0x1F (31).
    * Writing:
        * You can leave the extended axis values unmodified if your application does not use it.
        * If you are writing to a position register using Joint mode, do not panic if you do not see correct values.  Make sure you have the correct pose representation.

Below writes the CoDeSys **DINT** called `_diNumReg` to the robot's numeric register index 5
```
_stCipService.class := 16#6B; // numeric register (integer)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#05; // register index 5
_bGoodWrite := _Robot.bSetAttributeSingle(stCipService:=_stCipService,
                                            pbBuffer:=ADR(_diNumReg),
                                            uiSize:=SIZEOF(_diNumReg),
                                            psId:=ADR('Write R#5: '));
```

*
*
*

Below writes the CoDeSys **STRUCT** called `_stStringRegister` to the robot's string register index 5
**NOTE:** Writing to a robot string register must follow the format of a STRUCT made up of length (DINT) and a STRING.  Specify the length before writing!  See `Struct` folder for more details
```
_stCipService.class := 16#6D; // string register
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#05; // register index 5
_bGoodWrite := _Robot.bSetAttributeSingle(stCipService:=_stCipService,
                                            pbBuffer:=ADR(_stStringRegister),
                                            uiSize:=SIZEOF(_stStringRegister),
                                            psId:=ADR('Write SR#5: '));
```
Below writes the CoDeSys **STRUCT** called `_stPosReg2` to the robot's position register index 2
```
_stCipService.class := 16#7C; // position register (joint)
_stCipService.instance := 16#01; // single item
_stCipService.attribute := 16#02; // register index 2
_bGoodWrite := _Robot.bSetAttributeSingle(stCipService:=_stCipService,
                                            pbBuffer:=ADR(_stPosReg2),
                                            uiSize:=SIZEOF(_stPosReg2),
                                            psId:=ADR('Write PR#2: '));
```
Below writes the CoDeSys **STRUCT** called `_stBlockRegsInt` as a block of 10 registers to the robot numeric register starting with index 2
```
_stCipService.cipService := 16#33; // set attribute block
_stCipService.class := 16#6B; // numeric register (integer)
_stCipService.instance := 16#0A01; // 10 (16#0A) elements of group 1 (16#01)
_stCipService.attribute := 16#02; // register index 2
_bGoodWrite := _Robot.bGenericService(stCipService:=_stCipService,
                                        pbInBuffer:=ADR(_stBlockRegsInt),
                                        uiInSize:=SIZEOF(_stBlockRegsInt),
                                        psId:=ADR('Write 10 reg: '));
```

### But does it work?
Testing was done using a Raspberry Pi 3, with CoDeSys 3.5.16.0 runtime installed, to communicate with a Fanuc R-30iB Mate controller running on firmware `V8.30237 (12/7/2017)`.

**PS**: You will probably find this useful too ztoka