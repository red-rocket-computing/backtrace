Backtrace
=========

Copyright 2015, Red Rocket Computing, LLC, stephen@redrocketcomputing.com

Overview
--------

Backtrace is a very small (less 4.5K) stack unwinder designed for deeply embedded 
C applications running on the ARM Cortex-M family of microprocessors. Backtrace 
uses the unwind tables generated by the ARM GCC -funwind-tables option
(see https://gcc.gnu.org/onlinedocs/gcc-5.3.0/gcc/Code-Gen-Options.html#Code-Gen-Options).

Backtrace fills in a pre-allocated array of backtrace frames consisting of
the current function address, the calling site address and optionally a pointer
to the name of the function (if the GCC -mpoke-function-name option is used).

Backtrace provides the following API:

```c
typedef struct backtrace
{
	void *function;   /* Address of the current address */
	void *address;    /* Calling site address */
	const char *name;
} backtrace_t;

/* Unwind the stack from the current address, filling in provided
   backtrace bufffer, limited by size of the buffer, return a count
   of the valid frames */	
int backtrace_unwind(backtrace_t *backtrace, int size)

/* Return a pointer to the function name at the specified address or
   "unknown" if the function name string is not present.  The address
    can be the entry point for the function or any address within the
    function's scope. */
const char *backtrace_function_name(uint32_t pc);
```

ARM estimates the overhead for the unwind tables at 2.6% to 3.6%
(see /doc/IHI0038B_ehabi.pdf). Using the -mpoke-function-name option increases
the overhead by the sum of the length all function names plus 4 bytes per 
function.

License
-------

To facilate the use of this library in statically linked open and closed source 
embedded systems, this library is licensed under the terms of the 
[Mozilla Public License, V2.0.](http://mozilla.org/MPL/2.0)

Acknowledgments
---------------

I would like to thank the following people for providing clear insight and sample 
code helping me untangle this dark corner of the compiler world:

- Ken Werner of Linaro
- Mika Westerberg of Nokia

Dependencies
------------

Backtrace requires a compatible/working ARM cross compiler supporting the following built-ins:

```c
__builtin_return_address /* Get the return address for the current function */
__builtin_frame_address  /* Get the current frame pointer */
```

and is known to work with recent version of gcc
(see https://launchpad.net/gcc-arm-embedded).

In addition your linker scripts should export the following symbols:

```c
__exidx_start /* Pointing to the start of the unwind index table */
__exidx_end   /* Pointing to the end of the unwind index table */
```

Building
--------

For the default release version (-O3)

```shell
CD=path/to/project
make
```

To build a debug (-O0 -g) version of the library:

```shell
CD=path/to/project
make V=1 BUILD_TYPE=debug
```
	
The library can be found at 

	path/to/project/BUILD_TYPE/libbacktrace.a

The build location can be adjusted with OUTPUT_ROOT which must be an absolute path:

```shell
make OUTPUT_ROOT=/some/directory
```

Other build options can be adjusted in Makefile.common, Makefile.debug and Makefile.release.

Using The Unwinder
-------------------

```c
/*
 * Copyright 2015 Stephen Street <stephen@redrocketcomputing.com>
 * 
 * This Source Code Form is subject to the terms of the Mozilla Public
 * License, v. 2.0. If a copy of the MPL was not distributed with this
 * file, You can obtain one at http://mozilla.org/MPL/2.0/. 
 */

#include <stdio.h>
#include <stdint.h>
#include <backtrace.h>

#define BACKTRACE_SIZE 25

static int ping(int ball, backtrace_t *backtrace, int size);
static int pong(int ball, backtrace_t *backtrace, int size);

static void dump_backtrace(const backtrace_t *backtrace, int count)
{
	for (int i = 0; i < count; ++i)
		printf("%p - %s@%p\n", backtrace[i].address, backtrace[i].name, backtrace[i].address);
}

static int pong(int ball, backtrace_t *backtrace, int size)
{
	if (ball > 0)
		return ping(ball - 1, backtrace, size);
	else
		return backtrace_unwind(backtrace, size);
}

static int ping(int ball, backtrace_t *backtrace, int size)
{
	if (ball > 0)
		return pong(ball - 1, backtrace, size);
	else
		return backtrace_unwind(backtrace, size);
}

int main(int argc, char **argv)
{
	backtrace_t backtrace[BACKTRACE_SIZE];

	/* Play ball */
	int count = ping(10, backtrace, BACKTRACE_SIZE);

	/* Dump the backtrace */
	dump_backtrace(backtrace, count);

	/* All good */
	return 0;
}
```

To compile and link this sample using a recent GCC ARM Embedded compiler:

```shell
arm-none-eabi-gcc -mthumb -mcpu=cortex-m3 -I /path/to/include \
-fmessage-length=0 -fsigned-char -ffunction-sections \
-fdata-sections -std=c11 -mpoke-function-name -funwind-tables \
-fno-omit-frame-pointer ping-pong.c -o ping-pong.afx \
--specs=rdimon.specs --specs=nano.specs -L /path/to/lib -lbacktrace
```

To run ping-poing.afx with qemu:

```shell
> qemu-arm-static -cpu cortex-m3 ./ping-pong.afx
  pong - 10
  pong - 9
  pong - 8
  pong - 7
  pong - 6
  pong - 5
  pong - 4
  pong - 3
  pong - 2
  pong - 1
  pong - 0
  0x82f6 - ping@0x82f6
  0x8273 - pong@0x8273
  0x82e7 - ping@0x82e7
  0x8273 - pong@0x8273
  0x82e7 - ping@0x82e7
  0x8273 - pong@0x8273
  0x82e7 - ping@0x82e7
  0x8273 - pong@0x8273
  0x82e7 - ping@0x82e7
  0x8273 - pong@0x8273
  0x82e7 - ping@0x82e7
  0x834d - main@0x834d
  0x815b - unknown@0x815b
```
	

