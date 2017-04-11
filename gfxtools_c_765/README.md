In `gfxtools.c:765`:

```c
gfxbbox_t gfxline_getbbox(gfxline_t*line)
{
    gfxcoord_t x=0,y=0;
    gfxbbox_t bbox = {0,0,0,0};
    char last = 0;
    while(line) {
        if(line->type == gfx_moveTo) {    <--- line is pointing to an invalid address
            last = 1;

```

ToDo: I don't know where this value is corrupted. It is looks like a linked list:
```c
        x = line->x;
        y = line->y;
        line = line->next;
    }
```

But I don't know if it is a User After free or a memory corruption, if so maybe it is exploitable.
