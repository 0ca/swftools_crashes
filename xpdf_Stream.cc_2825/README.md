In `xpdf/Stream.cc:2825`:

```c
  do {
    // add a bit to the code
    if ((bit = readBit()) == EOF)
      return 9999;
    code = (code << 1) + bit;
    ++codeBits;

    // look up code
    if (code - table->firstCode[codeBits] < table->numCodes[codeBits]) {
      code -= table->firstCode[codeBits];
      return table->sym[table->firstSym[codeBits] + code];   <--- buffer overflow 
    }
  } while (codeBits < 16);

```

The variable `table->sym` is defined as:
```c
// DCT Huffman decoding table
struct DCTHuffTable {
  Guchar firstSym[17];          // first symbol for this bit length
  Gushort firstCode[17];        // first code for this bit length
  Gushort numCodes[17];         // number of codes of this bit length
  Guchar sym[256];              // symbols
};
```

So sym can only hold 256 bytes. But the variable used to access this array, `code`, is a short int. And parsing the PoC `code` gets the value: `65422` and the variable sym is overflown.

This is **NOT EXPLOITABLE**. Because it is a overflow reading from allocated memory. Maybe it could be use as a memory leak.
