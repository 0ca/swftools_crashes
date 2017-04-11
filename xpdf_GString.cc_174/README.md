
In `xpdf_GString.cc_174` :

```c
GString::~GString() {
  delete[] s; <---- s is not defined so it crashes
}
```

It comes because in `Function.cc.68` :

```c
 else if (funcType == 4) {
    func = new PostScriptFunction(funcObj, dict);
  }
```

the `PostScriptFunction` constructor does not finish properly filling all the values since at `Function.cc.1019`: 

```c
  //----- get the stream
  if (!funcObj->isStream()) {
    error(-1, "Type 4 function isn't a stream");
    goto err1;   <--- This is follow
  }
  str = funcObj->getStream();

  //----- parse the function
  codeString = new GString();  <--- codeString is initialized (Not reached in the PoC)
  ...
   err1:
  return;
}  
```

The execution goes to `err1` but still thinks that `CodeString` was assignated but it wasn't.

It is not a double free but a free to unitializated memory.

This could be **EXPLOITABLE** but it requires a lot of research and luck.

This bug could be use to create a use after free.

The concept is the next:
* First we create a valid PostScriptFunction and the the codeString in initialized correctly.
* Then we free the PostScriptFunction, and the codeString gets free.
* Then we allocate an object of the same size of codeString, objectX (We need to find this object).It will use the codeString free block.
* Then we create an invalid PostScriptFunction, it will use the first PostScriptFunction free block. Since it is invalid the codeString pointer is unitialized and it is pointing to the new object, objectX.
* Then we can do two things:
** Use the PostScriptFunction codeString as a objectX
** Free the PostScriptFunction, so objectX gets free and then we allocate another object, objectY, and we use this the objectX to try to execute code or do an arbitrary write.
