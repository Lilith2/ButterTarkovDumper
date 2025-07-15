# Docs

## Terminology I use to explain

- Anchor: A field, method, class name, or other lexical token you search for to find the desired structure in the dump file or in assembly-csharp. https://en.wikipedia.org/wiki/Lexical_analysis#Token

- Scope: The range a variable can be referenced. https://en.wikipedia.org/wiki/Scope_(computer_science)


## ProcessClassNames function

If you want to find methods of classes, add your structures in scope of this function.

### entity
Anchor, field or class name usually.

### variable
Dumper equals it to the real class name in Assembly-CSharp in SDK.cs ("ClassName" is typically used for field name)

### SetVariableStatus(variable);
Resets the finder module to target our anchor (entity).

### new StructureGenerator object
The parameter is the name you want the struct in sdk.cs

### nestedStruct.AddClassName() 
Should be first, it actually uses the finder module to get the method defined in string entity.

### nestedStruct.AddString() 
Add a string type field to the struct, defining the exact method name to target.

### structGenerator.AddStruct(nestedStruct)  
ProcessClassNames parameter object to add our nested structure to the main file.


## ProcessOffsets

If you want fields of a class.

### TypeDef name_here = default;
Variable in code that holds the return value from _dnlibHelper.FindClassWithEntityName()

### name
Name of the struct that gets written into SDK.cs

### SetVariableStatus(name); 
Resets the finder module to target our anchor. Parameter is the name variabe defined above.

### new StructureGenerator object
The parameter is the name variable defined above.

### const string ClassName
The actual class name from Assembly-CSharp.

### entity
Field name you need the offset of

### set the TypeDef variable from step 1 to the return of _dnlibHelper.FindClassWithEntityName()

### var offset
dynamic typed variable set to result of _dumpParser.FindOffsetByTypeName(ClassName, $"-.{MenuOperationClass.Humanize()}");

### call to nestedStruct.AddOffset(entity, offset);

### call to structGenerator.AddStruct(nestedStruct);


## Strange case of Anti-AFK SDK structs

For example, FindClassWithEntityName could have been reused but instead the coder used two var fMethod & var fField to setup for FindClassByTypeName on AFKMonitor structure.
Reading all the libraries, it's dealing with parsing opcodes in byte arrays and a custom row / collumn format in memory of the opcodes for each method. The returns are custom dnspy types that are very complex with poor naming conventions.

Looking in dnspy.exe at a deobfuscated Assembly-CSharp, I can plainly see all the data attempting to be found by the dumper code, although I can't determine why the functions returning opcode types were used or how they are specifically determining offsets. There is a shit-ton of abstraction and layers.

I will experiment with a simplier approach....

```c#
TypeDef MenuOperationClass = default;

{
    string name = "TarkovApplication";
    SetVariableStatus(name);

    StructureGenerator nestedStruct = new(name);
    const string ClassName = "EFT.TarkovApplication";
    string entity = "MenuOperation";

    {
        MenuOperationClass = _dnlibHelper.FindClassWithEntityName("StopAfkMonitor", DnlibHelper.SearchType.Method);
        var offset = _dumpParser.FindOffsetByTypeName(ClassName, $"-.{MenuOperationClass.Humanize()}");
        nestedStruct.AddOffset(entity, offset);
    }

    structGenerator.AddStruct(nestedStruct);
}

string AfkMonitorClassName = default;

{
    string name = "MenuOperation";
    SetVariableStatus(name);

    StructureGenerator nestedStruct = new(name);

    string entity;

    {
        entity = "AfkMonitor";

        var fMethod = _dnlibHelper.FindMethodByName(MenuOperationClass, "StopAfkMonitor");
        var fField = _dnlibHelper.GetNthFieldReferencedByMethod(fMethod);
        AfkMonitorClassName = fField.GetTypeName();
        var offset = _dumpParser.FindOffsetByName(MenuOperationClass.Humanize(), fField.Humanize());
        nestedStruct.AddOffset(entity, offset);
    }

    structGenerator.AddStruct(nestedStruct);
}

{
    string name = "AfkMonitor";
    SetVariableStatus(name);

    StructureGenerator nestedStruct = new(name);

    string entity;

    var fClass = _dnlibHelper.FindClassByTypeName(AfkMonitorClassName);

    {
        entity = "Delay";

        var fMethod = _dnlibHelper.FindMethodByName(fClass, "Start");
        var fField = _dnlibHelper.GetNthFieldReferencedByMethod(fMethod, 3);
        var offset = _dumpParser.FindOffsetByName(fClass.Humanize(), fField.Humanize());
        nestedStruct.AddOffset(entity, offset);
    }

    structGenerator.AddStruct(nestedStruct);
}
```

## How to Use

1. **Dump the Game**:
   - Use **Unispect** to dump the game data, I use DMA plugin for this from DER discord.
   - Save the dump file for processing.

2. **Prepare the Dumper**:
   - Open source code in your favorite editor, define the path variables in EFTProcessor.cs file.
   - Put dump file into directory as defined in file EFTProcessor.cs in variable DUMP_INPUT_PATH
   - Put Assembly-CSharp.dll file into directory as defined in file EFTProcessor.cs in variable ASSEMBLY_INPUT_PATH
3. **Run Dumper**
   - Should create the sdk.cs file in the path you've previously set.

## Requirements
- **Unispect** for initial game dumping.
