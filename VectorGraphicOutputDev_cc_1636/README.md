In `VectorGraphicOutputDev.cc:1636`:

```c
void VectorGraphicOutputDev::clearSoftMask(GfxState *state)
{
    if(!states[statepos].softmask)
        return;
    states[statepos].softmask = 0;
    dbg("clearSoftMask statepos=%d", statepos);
    msg("<verbose> clearSoftMask statepos=%d", statepos);

    if(!states[statepos].softmaskrecording || strcmp(this->device->name, "record")) {    <--- this->device is NULL
        msg("<error> Error in softmask/tgroup ordering");
        return;
    }
```

While parsing the PoC the `this->device` attribute is not initiallized and the program crashes dereferencing it.
 
This is not **EXPLOITABLE**.

ToDo: Why is this happening?
