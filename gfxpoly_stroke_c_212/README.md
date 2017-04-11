In `gfxpoly/stroke.c:212`:

```c
        lastx = points[pos].x = line->x;    <--- points is NULL!
        lasty = points[pos].y = line->y;
```

The varibale points is NULL so the program crashes dereferencing the pointer.
```c

    gfxpoint_t* points = malloc(sizeof(gfxpoint_t)*size);
    line = start;
    pos = 0;
```

When the variable size is very long malloc in the PoC:
```c
failed to allocate 0x0007f658bfc0 bytes
```
malloc doesn't allocate any memory, but the program doesn't contemplate that possibility and the crash happens.

This is not **EXPLOITABLE** in 64 bits.

In 32 bits there is an integer overflow calculating the requested allocating size. But it is not clear a user could control how to overflow the buffer.
