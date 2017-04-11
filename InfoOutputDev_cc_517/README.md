In `InfoOutputDev.cc:517`:

```c
char*getFontID(GfxFont*font)
{
    Ref*ref = font->getID();
    GString*gstr = font->getName();
    char* fname = gstr==0?0:gstr->getCString();
    char buf[128];
    if(fname==0) {
        if(font->getType() == fontType3) {
            sprintf(buf, "t3font-%d-%d", ref->num, ref->gen);
        } else {
            sprintf(buf, "font-%d-%d", ref->num, ref->gen);
        }
    } else {
        sprintf(buf, "%s-%d-%d", fname, ref->num, ref->gen);   <--- Buffer overflow when the formated string is larger than 128
    }
    return strdup(buf);
}
```

There is a buffer overflow when sprintf is call and the formatted string is larger than 128 bytes `char buf[128];`. When this happens it depends of the arguments.

For example with `ref->num=0` and `ref->gen=0` the buffer overflow occurs with a `fname` greater than 124.
```
%s-0-0
```

`fname` can only hold a maximum of 127 characters. 

So the overflow is limited. And it is **NOT EXPLOITABLE**>
