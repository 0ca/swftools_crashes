In `InfoOutputDev.cc:887`:
```c
void InfoOutputDev::type3D1(GfxState *state, double wx, double wy, double llx, double lly, double urx, double ury)
{
    if(-lly>current_type3_font->descender)    <--current_type3_font is not assigned, so it crashes
        current_type3_font->descender = -lly;
    if(ury>current_type3_font->ascender)
```

This is **NOT EXPLOITABLE**, since there is no way to control the value of `current_type3_font`.

The variable `current_type3_font` is allocated at:
```c
GBool InfoOutputDev::beginType3Char(GfxState *state, double x, double y, double dx, double dy, CharCode code, Unicode *u, int uLen)
{
...
    fontclass_clear(&fontclass);

    current_type3_font = fontinfo;
    fontinfo->grow(code+1);
...
```

But this function `InfoOutputDev::beginType3Char` is not called. So `current_type3_font` is never assigned.
