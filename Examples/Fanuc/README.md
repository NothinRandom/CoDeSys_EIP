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

    _stAttributeSingle  : CoDeSys_EIP.stAttributeSingle;
    _stLPOS             : stCartesianPosition;  // current linear position
    _stJPOS             : stJointPosition;      // current joint position
    _stPosReg1          : stCartesianPosition;  // position register
    _stPosReg2          : stJointPosition;      // position register

    _rNumReg            : REAL;                 // test numeric register
    _diNumReg           : DINT;                 // test numeric register
    _stStringRegister   : stStringRegister;     // test string register
END_VAR
```

In your code, toggle `bEnable` of _Robot to `TRUE`.  There is optional `bAutoReconnect` to re-establish session if terminated from idling (looks like robot controller never closes session).  You need to specify Unconnected Messaging only (Send RR Data) via `_Robot.bUnconnectedMessaging := TRUE` (see `Fanuc_RPi.project`)!

#### Data alignment:
Fanuc robot follows similar layout like Rockwell with 4 bytes alignment when we have gaps (not 4B divisible), so make sure you specify CoDeSys STRUCTs with `{attribute 'pack_mode' := '4'}` when appropriate.  Read the [CoDeSys pack mode](https://help.codesys.com/webapp/_cds_pragma_attribute_pack_mode;product=codesys;version=3.5.16.0).

#### Available services
Below is a table that describes possible operations

**NOTE:** Only get/set attribute single is currently available.  Check back later for other services
```
=====================
Service Code:
=====================
* 0x01 (1)      : get attribute all
* 0x02 (2)      : Set attribute all
* 0x0E (14)     : get attribute single
* 0x10 (16)     : set attribute single
* 0x32 (50)     : get attribute block
* 0x33 (51)     : set attribute block
=====================
Class Code:
=====================
* 0x04 (4)      : digital inputs/outputs

* 0x6B (107)    : numeric register integer
* 0x6C (108)    : numeric register real
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
* 0x01 (1)      : index of single item (if block items, then this is starting index)
```

#### Reading Values
**NOTE**:
* Possible arguments for `bGetAttributeSingle`/`bSetAttributeSingle`:
    * `stAttribute` (STRUCT) [**required**]: Struct element.
    * `pbBuffer` (POINTER TO BYTE) [**required**]: Pointer to input/output buffer.
    * `uiSize` (UINT): Size of input/output buffer (pbBuffer).
    * `psId` (POINTER TO STRING): Pointer to caller id (e.g. psId:=ADR('Write#1: ').
        * Useful for troubleshooting; output to _PLC.sError.
* Function returns TRUE on successful read/write
* Numeric value will be casted based on your request.
    * **Example:** Numeric register #5 has value of `11.111`.  If you request integer (class := 16#6B), then return value will be `11`.  If you request real (class := 16#6C), then return value will be `11.111`.
* String register holds up to 253 characters.
* **Position registers**:
    * UserFrame has value of 0x3F (63) and ToolFrame of 0x1F (31).
    * Reading:
        * If you read an uninitialized register, X/Y/Z/W/P/R/EXT1/EXT2/EXT3 (Cartesian) or J1/J2/J3/J4/J5/J6/J7/J8/J9 (joint) will return not a number (NaN).
    * Writing:
        * You can leave the extended axis values unmodified if your application does not use it.
        * If you are writing to a position register using Joint mode, do not panic if you do not see correct values.  Make sure you have the correct pose representation.

Below reads numeric register index 5, which is a **DINT** and writes to a CoDeSys **DINT** called `_diNumReg`
```
_stAttributeSingle.class := 16#6B;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#05; // register index 5
_bGoodRead := _Robot.bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_diNumReg),
                                        uiSize:=SIZEOF(_diNumReg),
                                        psId:=ADR('Read R#5: '));
```
Below reads numeric register index 5, which is a **REAL** and writes to a CoDeSys **REAL** called `_rNumReg`
```
_stAttributeSingle.class := 16#6C;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#0A; // register index 10
_bGoodRead := _Robot.bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_rNumReg),
                                        uiSize:=SIZEOF(_rNumReg),
                                        psId:=ADR('Read R#10: '));
```
Below reads string register index 5, which is a **STRUCT** and writes to a CoDeSys **STRUCT** called `_stStringRegister`
```
_stAttributeSingle.class := 16#6D;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#05; // register index 5
_bGoodRead := _Robot.bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_stStringRegister),
                                        uiSize:=SIZEOF(_stStringRegister),
                                        psId:=ADR('Read SR#5: '));
```
Below reads position register index 1, which is a **CARTESIAN** position and writes to a CoDeSys **STRUCT** called `_stPosReg1`
```
_stAttributeSingle.class := 16#7B;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#01; // register index 1
_bGoodRead := _Robot.bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_stPosReg1),
                                        uiSize:=SIZEOF(_stPosReg1),
                                        psId:=ADR('Read PR#1: '));
```
Below reads position register index 2, which is a **JOINT** position and writes to a CoDeSys **STRUCT** called `_stPosReg2`
```
_stAttributeSingle.class := 16#7C;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#02; // register index 2
_bGoodRead := _Robot.bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_stPosReg2),
                                        uiSize:=SIZEOF(_stPosReg2),
                                        psId:=ADR('Read PR#2: '));
```
Below reads the robot's position in **CARTESIAN** format and writes to a CoDeSys **STRUCT** called `_stLPOS`
```
_stAttributeSingle.class := 16#7D;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#01;
_bGoodRead := _Robot.bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_stLPOS),
                                        uiSize:=SIZEOF(_stLPOS),
                                        psId:=ADR('Read LPOS: '));
```
Below reads the robot's position in **JOINT** format and writes to a CoDeSys **STRUCT** called `_stJPOS`
```
_stAttributeSingle.class := 16#7E;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#01;
_bGoodRead := _Robot.bGetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_stJPOS),
                                        uiSize:=SIZEOF(_stJPOS),
                                        psId:=ADR('Read JPOS: '));
```
#### Writing Values
**NOTE**:
* Writing all data types follows the same format as read.

Below writes the CoDeSys **DINT** called `_diNumReg` to the robot's numeric register index 5
```
_stAttributeSingle.class := 16#6B;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#05; // register index 5
_bGoodRead := _Robot.bSetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
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
_stAttributeSingle.class := 16#6D;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#05; // register index 5
_bGoodRead := _Robot.bSetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_stStringRegister),
                                        uiSize:=SIZEOF(_stStringRegister),
                                        psId:=ADR('Write SR#5: '));
```
Below writes the CoDeSys **STRUCT** called `_stPosReg2` to the robot's position register index 2
```
_stAttributeSingle.class := 16#7C;
_stAttributeSingle.instance := 16#01;
_stAttributeSingle.attribute := 16#02; // register index 2
_bGoodRead := _Robot.bSetAttributeSingle(stAttributeSingle:=_stAttributeSingle,
                                        pbBuffer:=ADR(_stPosReg2),
                                        uiSize:=SIZEOF(_stPosReg2),
                                        psId:=ADR('Write PR#2: '));
```

### But does it work?
Testing was done using a Raspberry Pi 3, with CoDeSys 3.5.16.0 runtime installed, to communicate with a Fanuc R-30iB Mate controller running on firmware `V8.30237 (12/7/2017)`.

**PS**: You will probably find this useful too ztoka