Taintgrind: a Valgrind taint analysis tool
==========================================

2017-08-10 Support for Valgrind 3.13.0, x86\_linux and amd64\_linux

2015-10-06 Support for Valgrind 3.11.0, x86\_linux and amd64\_linux

2015-10-06 Highly experimental feature: SMT-libv2 output via --smt2=yes

2014-09-25 Support for client requests

2014-09-15 Support for Valgrind 3.10.0, x86\_linux and amd64\_linux

2013-12-20 Experimental support for 32-bit ARM, tested on Android 4.4 emulator with API 19

2013-11-18 Currently supporting: Valgrind 3.9.0, x86\_linux and amd64\_linux



Installation
------------

1. Download [Valgrind](http://valgrind.org) and build


		[me@machine ~/] tar jxvf valgrind-X.X.X.tar.bz2
		[me@machine ~/] cd valgrind-X.X.X
		[me@machine ~/valgrind-X.X.X] ./autogen.sh
		[me@machine ~/valgrind-X.X.X] ./configure --prefix=`pwd`/inst
		[me@machine ~/valgrind-X.X.X] make && make install

2. Git clone and build taintgrind


		[me@machine ~/valgrind-X.X.X] git clone http://github.com/wmkhoo/taintgrind.git
		[me@machine ~/valgrind-X.X.X] cd taintgrind 
		[me@machine ~/valgrind-X.X.X/taintgrind] ../autogen.sh
		[me@machine ~/valgrind-X.X.X/taintgrind] ./configure --prefix=`pwd`/../inst
		[me@machine ~/valgrind-X.X.X/taintgrind] make && make install

3. To compile examples in tests/


		[me@machine ~/valgrind-X.X.X/taintgrind] make check

A simple example
----------------

A simple example is tests/sign32.c

	#include "taintgrind.h"
	int get_sign(int x) {
	    if (x == 0) return 0;
	    if (x < 0)  return -1;
	    return 1;
	}
	int main(int argc, char **argv)
	{
	    int a = 1000;
	    // Defines int a as tainted
	    TNT_MAKE_MEM_TAINTED_NAMED(&a,4,"myint");
	    int s = get_sign(a);
	    return s;
	}

The TNT_MAKE_MEM_TAINTED_NAMED client request taints a, which is 4 bytes in size.

Compile with

	[../taintgrind] make check

Run with

	[valgrind-x.xx.x] ./inst/bin/valgrind --tool=taintgrind ~/sign32

The first tainted instruction should be

	0x804855A: main (sign32.c:14) | t19_9142 = LOAD I32 t17_9300 | 0x3e8 | t19_9142 <- a_1

The output of taintgrind is a list of Valgrind IR (VEX) statements of the form

	Address/Location | VEX-IRStmt | Runtime value(s) | Information flow

The first instruction indicates a byte (type I32, or int32\_t) is loaded from a into temporary variable t19\_9142. Its run-time value is 0x3e8 or 1,000. An instruction with no tainted variables will not have information flow. With debugging information, taintgrind can list the source location (sign32.c:14) and the variable name (a_1).
Only one run-time/taint value per instruction is shown. That variable is usually the one being assigned, e.g. t23\_1 in this case. In the case of an if-goto, it is the conditional variable; in the case of an indirect jump, it is the jump target. For loads and stores, we have simply chosen to print the data.
Details of VEX operators and IRStmts can be found in VEX/pub/libvex\_ir.h .
The 2 tainted if-gotos should come up as

	0x80484A4: get_sign (sign32.c:3) | IF t28_3680 GOTO 0x80484a6 | 0x0 | 0x1 | t28_3680
	0x80484B1: get_sign (sign32.c:4) | IF t6_14297 GOTO 0x80484b3 | 0x0 | 0x1 | t6_14297

As expected, the conditions are both false, and are thus 0.
Finally the return value of get\_sign should be

	0x80484BA: get_sign (sign32.c:5) | r8_13565 = 0x1 | 0x1 | 0x0 | 
	
See [Detecting a classic buffer overflow](https://github.com/wmkhoo/taintgrind/wiki/Detecting-a-classic-buffer-overflow)


Graph Visualisation
-------------------

Create a Graphviz dot file with e.g.

	> valgrind --tool=taintgrind tests/sign32 2>&1 | python log2dot.py > sign32.dot

Visualise the graph with

	> dot -Tpng sign32.dot -o sign32.png
![Example taint graph](/images/sign32.png)



Tainting file input
-------------------

	[me@machine ~/valgrind-X.X.X] ./inst/bin/valgrind --tool=taintgrind --help
	...
	user options for Taintgrind:
	    --file-filter=<full_path>   full path of file to taint [""]
	    --taint-start=[0,800000]    starting byte to taint (in hex) [0]
	    --taint-len=[0,800000]      number of bytes to taint from taint-start (in hex)[800000]
	    --taint-all= no|yes         taint all bytes of all files read. warning: slow! [no]
	    --tainted-ins-only= no|yes  print tainted instructions only [yes]
	    --critical-ins-only= no|yes print critical instructions only [no]

If the file-filter field is '\*', it is equivalent to --taint-all=yes.
Tainted instructions are really instructions where one or more of its input/output variables are tainted.
At the moment, critical instructions include loads, stores, conditional jumps and indirect jumps/calls. If --critical-ins-only is turned on, all other instructions are not printed.
The last two options control the output of taintgrind. If both of these options are 'no', then taintgrind prints every instruction executed. 
Run without any parameters, taintgrind will not taint anything and the program output should be printed.

Run Taintgrind with e.g.

	> valgrind --tool=taintgrind --file-filter=/path/to/test.txt --taint-start=0 --taint-len=1 gzip path/to/test.txt

See [Generating SMT Libv2 output](https://github.com/wmkhoo/taintgrind/wiki/Generating-SMT-Libv2-output)

Notes
-----

Taintgrind is based on [Valgrind](http://valgrind.org)'s MemCheck and [Flayer](http://code.google.com/p/flayer/).

Taintgrind borrows the bit-precise shadow memory from MemCheck and only propagates explicit data flow. This means that Taintgrind will not propagate taint in control structures such as if-else, for-loops and while-loops. Taintgrind will also not propagate taint in dereferenced tainted pointers.

Taintgrind has been used in [SOAAP](https://github.com/CTSRD-SOAAP/) and [Secretgrind](https://github.com/lmrs2/secretgrind).


License
-------

Taintgrind is licensed under GNU GPLv2.

