In `xpdf/FoFiTrueType.cc:1144`:
```c
  newTables = (TrueTypeTable *)gmallocn(nNewTables, sizeof(TrueTypeTable));
  j = 0;
  for (i = 0; i < nTables; ++i) {
    if (tables[i].len > 0) {
      newTables[j] = tables[i];
      newTables[j].origOffset = tables[i].offset;
      if (checkRegion(tables[i].offset, newTables[i].len)) {   <--- Typo, using newTables[i] instead of newTables[j]
        newTables[j].checksum =
            computeTableChecksum(file + tables[i].offset, tables[i].len);
```

The vulnerability is because of a typo using `i` instead of `j` to access the variable `newTables`.
```c
	checkRegion(tables[i].offset, newTables[i].len)
```

This makes that the program is reading outside the `newTables` variable limits.

This is **NOT EXPLOITABLE**.
