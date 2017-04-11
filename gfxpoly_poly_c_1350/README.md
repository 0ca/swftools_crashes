In `gfxpoly/poly.c:1350`:
```c
            case hevent_end:
                num_open--;
                if(num_open) {
                    open[num_open]->pos = e->h->pos;  <--- num_open could be a negative number
                    open[e->h->pos] = open[num_open];
                }
```

While parsing the PoC pdf2swf gets a negative number in the variable `num_open`. There is no validation in the program about the values this index can take.

The number gets decreased every time and event of type `hevent_end` is parsed:
```c
            case hevent_end:
                num_open--;
                if(num_open) {
```

So values below the array `open` gets overwritten with `e->h->pos`. It is not clear if the user controlls this value. But even if the user controls the value `e->h->pos`, this is used in the next line to read from the `open` array:
```c
	open[e->h->pos]
```

That makes the explotation to be very difficult, because the written value needs to be at the same time a good value to the next operation:
```c
open * sizeof(open) + e->h->pos  -> This needs to be a valid pointer to read from
```

This is **NOT EXPLOITABLE**.

