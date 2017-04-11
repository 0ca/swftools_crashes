In `InfoOutputDev.cc:879`:
```c
void InfoOutputDev::type3D0(GfxState *state, double wx, double wy)
{
    currentglyph->x1=0;  <--currentglyph is NULL
    currentglyph->y1=0;
    currentglyph->x2=wx;
    currentglyph->y2=wy;
}
```

This is **NOT EXPLOITABLE**, since there is no way to control the value of `currentglyph`.

The variable `currentglyph` is allocated at:
```c
GBool InfoOutputDev::beginType3Char(GfxState *state, double x, double y, double dx, double dy, CharCode code, Unicode *u, int uLen)
{
...
    if(!fontinfo->glyphs[code]) {
        currentglyph = fontinfo->glyphs[code] = new GlyphInfo();
...
```

But this function `InfoOutputDev::beginType3Char` is not called before. So `currentglyph` is never assigned.
