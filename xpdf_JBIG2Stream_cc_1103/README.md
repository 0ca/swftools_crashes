In `xpdf/JBIG2Stream.cc:1103`:

```c
class JBIG2CodeTable: public JBIG2Segment {
public:

  JBIG2CodeTable(Guint segNumA, JBIG2HuffmanTable *tableA);
  virtual ~JBIG2CodeTable();
  virtual JBIG2SegmentType getType() { return jbig2SegCodeTable; }
  JBIG2HuffmanTable *getHuffTable() { return table; }			<---- this variable is NULL
```

Parsing the PoC the function `getHuffTable` is call with a NULL value for the instance object, `this`. So when the code `this->table` is executed the program crashes.

The problem cames when this function is called from `xpdf/JBIG2Stream.cc:1988`:
```c
 huffRDYTable = ((JBIG2CodeTable *)codeTables->get(i++))->getHuffTable();
```

The codeTables array is initialized at `xpdf/JBIG2Stream.cc:1893`. But parsing a PoC an invalid segment reference is pass and the table is not fill.
```c
  codeTables = new GList();
  numSyms = 0;
  for (i = 0; i < nRefSegs; ++i) {
    // This is need by bug 12014, returning gFalse makes it not crash
    // but we end up with a empty page while acroread is able to render
    // part of it
    if ((seg = findSegment(refSegs[i]))) {
      if (seg->getType() == jbig2SegSymbolDict) {
        numSyms += ((JBIG2SymbolDict *)seg)->getSize();
      } else if (seg->getType() == jbig2SegCodeTable) {
        codeTables->append(seg);
      }
    } else {
      error(getPos(), "Invalid segment reference in JBIG2 text region");    <---  This line gets executed
    }
  }

```

This is not **EXPLOITABLE**.
