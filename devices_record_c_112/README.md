In `devices/record.c:112`:
```c
static void dumpLine(writer_t*w, state_t*state, gfxline_t*line)
{
    while(line) {
        if(line->type == gfx_moveTo) {		<--- line is pointing to invalid memory
            writer_writeU8(w, LINE_MOVETO);
            writer_writeDouble(w, line->x);
            writer_writeDouble(w, line->y);
```

This bug is related with `gfxtools_c_765.pdf` the root cause should be the same.

ToDo: Analyze root cause.

IT doesn't look exploitable, but it depends about how the variable `line` gets modified.
