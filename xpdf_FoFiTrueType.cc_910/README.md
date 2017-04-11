In `xpdf/FoFiTrueType.cc:910`:
```c
  // check for an incorrect cmap table length
  badCmapLen = gFalse;
  cmapLen = 0; // make gcc happy
  if (!missingCmap) {
    cmapLen = cmaps[0].offset + cmaps[0].len;     <--- cmaps is NULL
    for (i = 1; i < nCmaps; ++i) {

```

The variable `cmaps` is not allocated so the program crashes parsing the PoC because a NULL dereference.

`cmaps` is allocated at the function `FoFiTrueType::parse()` 
```c
  // read the cmaps
  if ((i = seekTable("cmap")) >= 0) {
    pos = tables[i].offset + 2;
    nCmaps = getU16BE(pos, &parsedOk);
    pos += 2;
    if (!parsedOk) {
      return;
    }
    cmaps = (TrueTypeCmap *)gmallocn(nCmaps, sizeof(TrueTypeCmap));   <--The table is allocated here
    for (j = 0; j < nCmaps; ++j) {
      cmaps[j].platform = getU16BE(pos, &parsedOk);
      cmaps[j].encoding = getU16BE(pos + 2, &parsedOk);
      cmaps[j].offset = tables[i].offset + getU32BE(pos + 4, &parsedOk);
      pos += 8;
      cmaps[j].fmt = getU16BE(cmaps[j].offset, &parsedOk);
      cmaps[j].len = getU16BE(cmaps[j].offset + 2, &parsedOk);
    }
    if (!parsedOk) {
      return;
    }
  } else {
    nCmaps = 0;   <---If the "cmap" is not found then nCmaps is 0
  }

```

In the PoC the variable `cmap` is not present so the `cmaps` variable is not allocated and `nCmaps` is set to 0.

But the program crashes when execute the lines shown at the beggining:
```c
  if (!missingCmap) {
    cmapLen = cmaps[0].offset + cmaps[0].len;     <--- cmaps is NULL
```

This is **NOT EXPLOITABLE**.
