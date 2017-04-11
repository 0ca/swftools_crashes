# Summary
There is a Use-After-Free, UAF, vulnerability in all the versions of pdf2swf, a software belonging to the [suite swftools](http://www.swftools.org/).

The vulnerability happens because of an incorrect handling of the graphic state while processing a Form XObject. This makes a Font resource used at a Form XObject to be freed after the XObject is processed, but the font object is still being used.

If an attacker is able to overwrite the freed font with controlled malicious data, he **could redirect the execution** when function pointers inside the font are called.

# Technical details
## Root cause
> Form XObjects is a way of describing objects (text, images, vector elements,…) within a PDF file. It is meant for repetitive objects that only get stored once in a PDF but referenced multiple times. In a way, Form XObjects are similar to a mini-PDF embedded inside the main PDF document
https://www.prepressure.com/pdf/basics/form-xobjects

In the following lines it is shown a Form XObject object following the PDF file format:
```ps
6 0 obj
<< 
        /Type /XObject 
        /Subtype /Form 
        /BBox [0 0 0 0] 
        /Resources 
        << 
                /Font << 
                        /F2 4 0 R 
                >> 
        >> 
        /Length 15
>> 
stream
/F2 0 Tf
(Text) Tj
q
endstream
endobj
```

A PDF object is composed of a dictionary with a set of variables and a stream with the content of the object. In the previous example, we see that the Form XObject is using a Font referenced with the name `/F2` defined in the object 4. Inside the stream we can see a set of PostScript operators. (More info: [Appendix A: Operator Summary](http://www.adobe.com/content/dam/Adobe/en/devnet/acrobat/pdfs/pdf_reference_1-7.pdf)). The first operator `Tf` sets the font and the size used for the next operands. `Tj` shows the text `Text`. And finally the operator `q` saves the graphic state.

The operator `q` and its opposite `Q` save and restore the graphic state. These operands are used to temporally use a different graphics settings (different Font type, size or color) for the text representation. Typically, the operators `q` and `Q` are balanced, and every time the state is saved there is an operator to restore the state before the Form XObject finishes.

The way to display an already defined Form XObject is using the PostScript operator `Do`. For example we could use the following object as Content of a PDF page to display the Form XObject `/XT5`, previously defined in the resource section:
```ps
3 0 obj
<< 
        /Type /Page 
        /Parent 2 0 R 
        /Resources
        <<
                /XObject 
                << 
                        /XT5 6 0 R 
                >> 
        >>
        /Contents 7 0 R 
>>
endobj

7 0 obj
<<
        /Length 7
>> 
stream
/XT5 Do
endstream
```

When pdf2swf renders a Form XObject with the `Do` operand, the state (representing the current Font, size, etc...) is first saved and after the XObject is shown the state is restored. Snippet at `lib/pdf/Gfx.cc:3781`:
```c
void Gfx::doForm1(Object *str, Dict *resDict, double *matrix, double *bbox,
                  GBool transpGroup, GBool softMask,
                  GfxColorSpace *blendingColorSpace,
                  GBool isolated, GBool knockout,
                  GBool alpha, Function *transferFunc,
                  GfxColor *backdropColor) {
  Parser *oldParser;
  double oldBaseMatrix[6];
  int i;

  // push new resources on stack
  pushResources(resDict);

  // save current graphics state
  saveState();
...
...
  // restore graphics state
  restoreState();

  // pop resource stack
  popResources();

  if (softMask) {
    out->setSoftMask(state, bbox, alpha, transferFunc, backdropColor);
  } else if (transpGroup) {
    out->paintTransparencyGroup(state, bbox);
  }

  return;
}
``` 

The function `popResources`, takes out and deletes the resources used by the XObject calling to the GfxResources destructor:
```c
GfxResources::~GfxResources() {
  if (fonts) {
    delete fonts;   <--- All fonts freed
  }
  xObjDict.free();
  colorSpaceDict.free();
  patternDict.free();
  shadingDict.free();
  gStateDict.free();
}
```

Using the operator `q` inside a XObject it is possible to add an additional state that it is never restored by `Q`. So when the Form XObject finishes, the resources, like the current font (/F2), are freed, but the restored state is still using the freed font.

It would be possible to use the freed font with the following Content page object:
```ps
7 0 obj
<<
        /Length 16
>> 
stream
/XT5 Do
(Text) Tj
endstream
```

Remember that `Tj` is the operator to display text. And the font used to do so it would be the freed font.

If an attacker is able to overwrite the freed font with controlled malicious data, he *could redirect the execution* when function pointers inside the font are called.

## How to overwrite the freed font
After the XObject is displayed with the `Do` operator and before the font is used, p.e. showing text, an attacker has a chance to do a set of allocations and replace the freed font object. This could be done making use of the pdf2swf Form parser, and the lexer.

As we saw before, pdf2swf is parsing every PostScript operator to render the PDF. This is performed with the xpdf 3.02 library (an old version) at the files `Parser.cc` and `Lexer.cc`. What we want is to replace the recently freed font object, so we need to know its size:
```c
sizeof(GfxFont) -> 0x11F0
```

The next step is to look for a way to force pdf2swf to do an allocation of the desired size and control the content of this block.

This could be done using a PDF hex string. The pdf format supports the representation of a string as a hex string following the next syntax:
```
<48 65 6c 6c 6f>  <- A hex string representing Hello
```

Inside the pdf2swf the hexstrings are stored as GString objects (`lib/pdf/xpdf/GString.cc`). A GString allocates memory is an exponential way, every time a string need to be resize it is multiplying per 2 the previous size and reallocating memory. So, having a big string it is possible to coerce GString to perform an allocation of 0x1000 bytes. This allocation is close enough to our 0x11F0 desired size, and the recently freed object will be reused by GString to stored the string the user provided.

This hex string format allows an attacker to represent any ascii character in memory form `0x00` to `0xff`.

## How the execution can be redirected
As we mentioned earlier, the way to redirect the execution is to overwrite a function pointer inside the freed font. 

This font is an instance of the class `Gfx8BitFont` that inherits from the class` `GfxFont`. These classes are defined at `lib\xpdf\GfxFont.h`. 

How to identify is a class has function pointers? We can take a look at the virtual functions defined in the class. When a class has virtual methods a virtual table at the beginning of the object is created with a list of all the pointer to the virtual functions the class implements. Some of the virtual methods of the classes `Gfx8BitFont` and `GfxFont` are shown in the following lines:
```c
  virtual GBool isCIDFont() { return gFalse; }
  virtual int getWMode() { return 0; }
  virtual int getNextChar(char *s, int len, CharCode *code,
                          Unicode *u, int uSize, int *uLen,
                          double *dx, double *dy, double *ox, double *oy) = 0;
  virtual CharCodeToUnicode* getCTU() = 0;
...
  virtual ~Gfx8BitFont();
```

If one of these methods are called after the font object is freed, then a function pointer obtained from a freed memory area will be called. But if somehow an attacker can allocate memory to replace the font object then the attacker data will be used to perform this call.

One of these methods, `isCIDFont` is called inside the function `doUpdateFont` (xpdf/SplashOutputDev.cc:1033) invoked when a text is displayed with the operator `Tr` at the function `opShowText`.
```c
    } else if (!(fileName = gfxFont->getExtFontFile())) {

      // look for a display font mapping or a substitute font
      if (gfxFont->isCIDFont()) {
        if (((GfxCIDFont *)gfxFont)->getCollection()) {
          dfp = globalParams->
                  getDisplayCIDFont(gfxFont->getName(),
                                    ((GfxCIDFont *)gfxFont)->getCollection());
        }
      }
```

The method `isCIDFont` is at the offset 0x10 inside the virtual table. But it is important to mention that the font object doesn't store this table, it only contains a pointer to the virtual table. So even replacing the freed font with controlled data, an attacker would need to allocate and know/predict the address where he writes a fake virtual table. To do so the attacker need to do heap spray to write enough data in the heap that he can assure that a predicted address is allocated with his data.

## How it is possible to do heap spray
In a previous section we explained how it is possible to use a hex string as a primitive to allocate controlled memory. But to do heap spray we need a primitive that allows us to do as many allocations as we want and, very important, these allocation needs to stay allocated, if they are freed every time we do another allocation, the same block would be used over and over again.

A first attempt we made it was to use as many hex strings as needed. But pdf2swf has some limitations about how many arguments an operator can have. The next code is an example of two arguments of type hex string passed to the operator `Tj` (display text). 
```ps
<48 65 6c 6c 6f> <48 65 6c 6c 6f> Tj
```
But pdf2swf has a limit of 33 arguments. So we could only do 33 allocations before an error is raised. 

But there is another way to do it without any limit. The PostScript format supports arrays, defined between brackets. So we can define an array with as many hex strings as we want as the argument to a `Tj` operator:
```ps
[<48 65 6c 6c 6f> <48 65 6c 6c 6f> ...] Tj
```

With enough strings of 0x1000 bytes it is possible to spray the heap with controlled data. So we could use this to spray a fake virtual table that will contain the function pointer called.

Different tests in Ubuntu and CentOS shown us that 10,000 allocations of 0x1000 bytes are enough to predict that the address 0x3000000 will be used as part of the heap to allocate the data. So that the address that could be use as a fake virtual table.

The PDF with the 10,000 hex strings has a size of ~80MB. In theory the PDF could be compressed and the size could be reduce a lot. But all the tests perfomed failed generating a valid compressed stream. It looks like the pdf2swf implementation for FlateDecode and LZWDecode is not compatible with zlib and some lzw libraries.

## How to execute whatever an attacker want
The pdf2swf is compiled with the following protections:
```
Protection	Enabled
----------	-------
ASLR		Yes
NX			Yes
RELRO		No
Canary		No
Seccomp 	No
PIE			No
```

Since NX is enabled we can't directly execute as a code the we are writing in the heap or the stack. We need to do ROP. But ASLR is enabled so we can't know where the different libraries are going to be loaded. Luckily PIE, Position Independent Executable, is not enabled, so it is possible to use the binary itself to create a ROP chain that jumps between different gadget until the desired code it is executed.

A first version of the ROP Chain was created to do a call to `system()` with a controlled argument. The steps are the next:
* Find a stack pivot so the stack points to our data
* POP from the stack the argument for the system call
* CALL system

A second version, more sophisticated, was done to extract from the pdf2swf memory the output name of the flash animation that is being generated. So the attacker can execute a set of command and the results of these commands will be written at the output file.

 The ROP chain works as follows:
* Find a stack pivot so the stack points to our data
* Find the output name in the pdf2swf memory
* POP from the stack the rest of the arguments for sprintf: The command to execute, the output name and the result buffer
* CALL sprintf
* POP from the stack the formatted buffer with the final command
* CALL system
* CALL exit to finish without raise a SIGSEV

A more detailed explanation about the ROP chain and details about the exploit can be found at the comments of the exploit itself: [exploit_pdf2swf0.9.0h_uaf.py](https://github.com/illera88/Oceanic/blob/master/swftools/crashes/InfoOutputDev_cc_567/exploit_pdf2swf0.9.0h_uaf.py)
