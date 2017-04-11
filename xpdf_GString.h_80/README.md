In `xpdf/GString.h:80`:

```c
  // Get C string.
  char *getCString() { return s; }     <---The code is accesing GString->s
```

The method belongs to the GString class. And the crashes happens because the code is accesing `GString->s` but `GString` is NULL.

The call comes from:
```c
DisplayFontParam *GFXGlobalParams::getDisplayFont(GString *fontName)
{
    msg("<verbose> looking for font %s", fontName->getCString());   //The compiler ommits this call

    char*name = fontName->getCString();
```

During the parsing of the PoC the variable `fontName` gets the value NULL. And the code doesn't check if the `fontName` is a correct string.

This is **NOT EXPLOITABLE**. It is a NULL dereference.
