In `VectorGraphicOutputDev.cc:1721`:
```c
    if(!config_textonly) {
        this->device->fillbitmap(this->device, line, belowimg, &matrix, 0);
    }
```

`this->device` is NULL so a NULL dereference happens. 

`this->device` is set at the beggining of the same function `VectorGraphicOutputDev::clearSoftMask`:
```c
    this->device = states[statepos].olddevice;
```

And it is here when it takes the NULL value.

This is **NOT EXPLOITABLE**.

ToDo: Find the root cause. Why olddevice is not initiallized?
