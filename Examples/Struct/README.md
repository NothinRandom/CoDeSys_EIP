# CoDeSys (STRUCT) <-> Rockwell (UDT)

Reference Rockwell UDTs (*.L5x)
Reference CoDeSys structs (*.txt)

Rockwell data is aligned on 4 bytes, so the CoDeSys structs need to declare bytes packing mode as `{attribute 'pack_mode' := '4'}`.  Else, you would need to pad the data with additional bytes.  There are some gotchas with string and bits, so take a look at the example structs.  I promise there is a pattern...well, kind of.

For `STRING` I have noticed so far is if the Rockwell string has odd/even length.  For example, a CoDeSys string that can hold 23 max characters (STRING[23]) consumes 24 total bytes since there is a terminatioon character.  However, it looks like a Rockwell string SINT[24] doesn't use termination char, but SINT[23] will pad 1 byte to keep it even.  Need a Rockwell expert to chime in on this topic.

For `BIT` alignment of at least 2 bytes is required if the multiple bits can not fill up a byte. Look at `udtBit.L5X` and `stBitTest.txt`

If you are unsure about how to convert Rockwell UDT to CoDeSys struct, then the best way is to view or export the format.  For example, Rockwell STRING24 UDT has 24 bytes.  However, if you create a CoDeSys string[24], then you actually use 25 total bytes, and the data types following the string will be incorrectly offset if you organize structs with mixed data types.


Rockwell UDT
```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<RSLogix5000Content SchemaRevision="1.0" SoftwareRevision="32.02" TargetName="STRING24" TargetType="DataType" ContainsContext="true" ExportDate="Fri Jul 10 10:17:10 2020" ExportOptions="References NoRawData L5KData DecoratedData Context Dependencies ForceProtectedEncoding AllProjDocTrans">
<Controller Use="Context" Name="EIP_Test">
<DataTypes Use="Context">
<DataType Use="Target" Name="STRING24" Family="StringFamily" Class="User">
<Members>
<Member Name="LEN" DataType="DINT" Dimension="0" Radix="Decimal" Hidden="false" ExternalAccess="Read/Write"/>
<Member Name="DATA" DataType="SINT" Dimension="24" Radix="ASCII" Hidden="false" ExternalAccess="Read/Write"/>
</Members>
</DataType>
</DataTypes>
</Controller>
</RSLogix5000Content>
```

CoDeSys equivalent struct
```
{attribute 'pack_mode' := '4'}

// NOTE: extremely important to have your bytes aligned correctly,
//      so read up on pack mode e.g. "{attribute 'pack_mode' := '0'}"
//      https://help.codesys.com/webapp/_cds_pragma_attribute_pack_mode;product=codesys;version=3.5.16.0

// as you can see here, there's a gotcha
// if you set {attribute 'pack_mode' := '0'}
// then you would need the extra padding
TYPE stString24 :
STRUCT
    len     : UDINT;
    text    : STRING[23];
    //pad   : ARRAY[1..3] OF BYTE;
END_STRUCT
END_TYPE
```

Data is incorrectly offset for `msgReady` when you pack into a mixed struct with declaration of string[24]:
```
{attribute 'pack_mode' := '4'}

// NOTE: extremely important to have your bytes aligned correctly,
//      so read up on pack mode e.g. "{attribute 'pack_mode' := '0'}"
//      https://help.codesys.com/webapp/_cds_pragma_attribute_pack_mode;product=codesys;version=3.5.16.0

// as you can see here, there's a gotcha
// if you set {attribute 'pack_mode' := '0'}
// then you would need the extra padding
TYPE stAlarmEvent :
STRUCT
    sTest1                  : stString24;
    sTest2                  : stString24;
    sTest3                  : stString24;
    sTest4                  : stString24;
    sMessage                : stString;
    rValue                  : REAL;
    sTest5                  : stString24;
    //ZZZZZZZZZZudtAlarmEv7 : SINT;
    bReady                  : BIT;
END_STRUCT
END_TYPE
```