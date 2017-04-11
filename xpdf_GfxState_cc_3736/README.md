In `xpdf_GfxState_cc_3736`:

```c
GfxState::~GfxState() {
  int i;

  if (fillColorSpace) {
    delete fillColorSpace;
  }
  if (strokeColorSpace) {
    delete strokeColorSpace;   <--- Use After Free
  }
  if (fillPattern) {
    delete fillPattern;
  }
```

The variable `strokeColorSpace` is initiallizated at the GfxState constructor at `xpdf_GfxState_cc_3760`:
```c
// Used for copy();
GfxState::GfxState(GfxState *state) {
  int i;

  memcpy(this, state, sizeof(GfxState));
  if (fillColorSpace) {
    fillColorSpace = state->fillColorSpace->copy();
  }
  if (strokeColorSpace) {
    strokeColorSpace = state->strokeColorSpace->copy();		<--- Variable initiallized in the GfxState constructor
  }
```

But this variable could be freed if the PDF has certain properties (p.e. in the PoC). ToDo: understand when it happens and why. 
This happens at ``:
```c

```

And then when the GfxState is destroyed the `strokeColorSpace` is deleted again:
```c
GfxState::~GfxState() {
...
  if (strokeColorSpace) {
    delete strokeColorSpace;
  }
...
```

But to find the destructor the strokeColorSpace vtable is accesed and the second function in the vtable is called.
```c
vtable structure:
+0:	Constructor address
+8:	Destructor address
* 64 bits
```

If between the first delete and the second one is possible to allocate memory and control its content we could take control of the execution.

We need to allocate two blocks of memory. And we need to predict/know the address of one of them. The idea is the first block will use the space free by the `strokeColorSpace`. And the first 8 bytes, the old vtable, will point to the second block. In the second block we need to control the offset 8, the expected destructor address.

Things ToDo:
* Is it anything parsed between the first and the second free?
* What is the size of the freed object?
```
print sizeof(GfxDeviceGrayColorSpace)
$18 = 8
```

* Are there any allocation primite we could use?
* Can we control the first 8 bytes of that primitive?
* Can we predict a memory address with data we control?


If the answer to all that question is yes, then this vulnerability is exploitable.

The current PoC is only freeing the address. After that the malloc implementation is writing an address at offset 0, the old vtable, and this address is used to jump to the vtable, add 8 bytes and call that address. As a result the program finish with a  SIGSEGV executing 0x00000000 00000021.

Being that 21 some number related with the malloc headers.
