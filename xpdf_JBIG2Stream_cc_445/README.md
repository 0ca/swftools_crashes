In `xpdf/JBIG2Stream.cc:445`:

```c
void JBIG2HuffmanDecoder::buildTable(JBIG2HuffmanTable *table, Guint len) {
...
...
  table[i++].prefix = prefix++;
  for (; table[i].rangeLen != jbig2HuffmanEOT; ++i) {    <--- heap-buffer-overflow reading variable table
    prefix <<= table[i].prefixLen - table[i-1].prefixLen;
    table[i].prefix = prefix++;
  }
```

In the PoC the argument `len` to the function is 0. In those cases the table only has one element as it is allocated in `xpdf/JBIG2Stream.cc:2018`:
```c
    huffDecoder->buildTable(runLengthTab, 35);
    symCodeTab = (JBIG2HuffmanTable *)gmallocn(numSyms + 1,
                                               sizeof(JBIG2HuffmanTable)
```

But in the first snipset of code, the variable i is incrementated and `table[1]` is accessed, reading beyond the variable limits.

Without ASAN the program crashes in another place:
```c
JBIG2HuffmanDecoder::reset()
```

This happens because this line `table[i].prefix = prefix++;` is overwritting the object `JBIG2HuffmanDecoder` and when the first method is called `reset` the program crashes.

This is **NOT EXPLOITABLE**.
