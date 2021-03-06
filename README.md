# learnasm370

The following repo has programs from the Bill Qualls assembler book 
**Mainframe Assembler Programming**, ISBN 0-471-24993-9 using the 
open source z390 assembler instead of PC/370.

https://www.billqualls.com/assembler/

This is not a strict coverage of every exercise in the book, but the 
programs I wrote as I read the book. Most of the major items are covered
and I may do a straight conversation at some point in the future. 

At this point its just notes. **YMMV**

## Why?

The book uses the PC/370 Assembler. This is a very old program and is no longer actively 
maintained. I had issues running the program using Windows 10.

I decided to try completing the book using the more modern tool z390.
z390 is written in Java which means it will run on platforms with a Java runtime available.
It was created by the same author as PC/370 (thanks Don Higgins). 

Mainframe assembler programming is still an important skill used today and in short demand.

## Install

This document assumes you have a Java SDK and Git client installed on your system.

### Download the latest release of z390

http://www.z390.info/

z390_v1703.zip (2020-11-19)

Unzip the folder onto you system.

* For Windows, choose a location that makes sense for your system.
* For MacOS, a good location is in ~/lib. If the folder does not already exist, create it.

### Clone this repo

Use a location where you normally work on code.

    git clone https://github.com/adelosa/learnasm370.git

### Config and run!

Open the newly cloned folder in your coding editor (see below for VSCode setup)

* For Windows, edit the `asmlg.bat` and change the variable `z390_dir` to point to your z390 installation folder.
* For MACOS, create a symbolic link in your path to the `perl/z390.pl` script in your z390 install folder. The link should be called z390. 

    `ln -s ~/lib/z390/perl/z390.pl ~/bin/z390`

You can now use the `asmlg.bat` or `asmlg.sh` scripts to assemble, link and run the assembler programs.

    win>   asmlg.bat hello.mlc

    macos> ./asmlg.sh hello.mlc

## Using Visual Studio code with IBM Z Open Editor

IBM provides an extension to Visual Studio Code that supports the editing of HLASM assembly programs.
I have found this editior very good and it works across Windows and MacOS.
It also supports macro expansion and validation although this requires connection to a real mainframe 
via Zowe. Even without this, its still a good editior experience.

You will need to update the IBM Z Open Editor extension config to recognise the file extensions used 
by z390. They are .MLC for programs and .MAC for macros.

I use the terminal within the editor to run the jobs to compile, link and run the program.

## Differences between PC/370 vs z390

PC/370 and z390 are not exactly the same. Most of the differences are in the macros
that are provided as part of the assembler. When I first started, I tried using the PC/370
macros with the z390 assembler but I found this problematic.

So I gave up and decided to port them.

This is not difficult once you know what the differences are.

The following section details where you will have issues with what is in the book and how to fix it using z390.

### Replace macro `REGS` with `EQUREGS`

z390 does not have the REGS macro that provides mapping of registers to R codes.

It does supply EQUREGS which provides the same functionality.

### Replace macro `BEGIN` with `SUBENTRY`

    BEGIN    BEGIN

BEGIN macro should be changed to SUBENTRY macro

    BEGIN    SUBENTRY

### `WTO` macro works differently

The WTO macro is used to write messages to the console (write to operator)


             WTO   MESSAGE
    ...
    MESSAGE  DC    C'Hello world!'

This will not work in z390. You will receive an error stating TEXT option not supported. Instead, recode as follows.

             WTO   'Hello world!'

If you want to display a part of memory, you will need to format a WTOBLOCK

             MVC   WTOTEXT(76),VALUE
             WTO   MF=(E,WTOBLOCK)
    ...
    WTOBLOCK EQU   *
             DC    H'80'         * For WTO, length of WTO buffer...
             DC    H'0'            should be binary zeroes...
    WTOTEXT  DS    CL76          * What will be displayed on console         

### File processing works differently

There is substantial differences in how PC/370 macros deal with files vs Z390 macros.

For working on the PC, there are some simplifications dealing with line endings in z390 that make sense to use.

#### `OPEN` macro

With z390, you need to specify if you will be opening a file for input or output

             OPEN  (TEACHERS,INPUT)
             OPEN  (REPORT,OUTPUT)

#### `DCB` macro

The following lines demonstrate the definition of an input (TEACHERS) and output (REPORT) files

    TEACHERS DCB   LRECL=27,RECFM=FT,MACRF=R,EODAD=ATEND,                  X
                   DDNAME=TEACHER,RECORD=IREC
    REPORT   DCB   LRECL=60,RECFM=FT,MACRF=W,                              X
                   DDNAME=REPORT

The `RECFM=FT` indicates a fixed length ASCII file and handles line endings by ignoring them.
There is also an `RECFM=VT` which operates in a similar way for variable length files.

The other difference is what the `DDNAME` option represents. On PC/370, the actual file name would be passed. For z390, this is an environment variable name that provides the filename.

In windows, you can set environment variables using the `SET` command.

    SET TEACHER=c:\teacher.dat

#### Removal of line ending/ASCII specific logic

Because of the PC file handling support in z390, you can remove any instructions in the programs that deal with ASCII and line endings. 

You can pretend you are reading a normal file on the mainframe with no line endings.

What this means:

File layouts don't need to allocate 2 bytes for line endings

    DS    CL2         28-29 PC/370 only - CR/LF`

No need for instructions that convert between ASCII and EBCDIC

    OI    TEACHERS+10,X'08'        PC/370 ONLY - Convert all
                                  input from ASCII to EBCDIC

No need to move line endings explicitly

    MVC   OCRLF,WCRLF               PC/370 ONLY - end line w/ CR/LF

### By default, overflows will generate abends

In chapter 7, the book discusses Packed decimal addition (`AP` instruction) and 
how to deal with overflows - that is where the result of an instruction
does not fit into the output memory.

If you just run the example, instead of getting to the `BO` (Branch overflow) instruction
you will receive a S0CA abend which is a Decimal Overflow Exception.

I am not sure why this behavior is different - it is possibly an enhancement to
later versions of assembler/MVS. In any case, you will need to explicitly turn off
the trap to mimic the behavior of the book.

I used the `ESPIE` macro to create an exit when a decimal overflow is encountered.

See https://www.ibm.com/support/knowledgecenter/SSLTBW_2.2.0/com.ibm.zos.v2r2.ieaa600/iea3a6_Using_the_ESPIE_macro.htm

You can also refer to the `pacdec.mlc` module for an example of its usage as per the book.

The macro seems to call the exit as a `BAS`, which means calling `RETURN` from the exit 
will return you to the instruction immediatly after the overflow occured. 

The bitmask for the branch is not set which means you cannot use the `BO` instruction. 
I set a character flag in the exit to indicate an overflow has occured and then do a standard 
branch based on its value (`CLC`).

## Other hints

* Keep your program file names to 8 characters. Otherwise, the assembler won't run your program.
* You can find a guide to the z390 macros here - http://www.z390.org/z390_Macro_and_Copybook_Documentation_Index.pdf