In `xpdf/Stream.cc:2270`:

```c
            // pull out the current values
            p1 = &frameBuf[cc][(y1+y2) * bufWidth + (x1+x2)];
            for (y3 = 0, i = 0; y3 < 8; ++y3, i += 8) {
              data[i] = p1[0];    <--- p1[0] pointing outside the allocated memory
              data[i+1] = p1[1];
              data[i+2] = p1[2];
              data[i+3] = p1[3];
              data[i+4] = p1[4];
```

The bug is a heap-buffer-overflow reading outside the variable `frameBuff`.

It looks like the problem comes form the copy to p1 and the use of these calculations:
```c
p1 = &frameBuf[cc][(y1+y2) * bufWidth + (x1+x2)];
```

Without ASAN the program crashes inside malloc. What makes me thing that a heap corruption is happening. But I don't know exactly how.

ToDo: 
* Why the overflow is happening
* When the heap corruption is happening
* Is it possible to control the heap corruption?

