---
layout: post
title:  "Getting the minimum version of glibc for a binary"
date:   2015-12-12 00:00:00
categories: golang cgo C dependencies glibc kernel linux
---

I ran into a problem recently where a cgo binary built on one version of Linux (with a 2.6.32 kernel) wouldn't run on a machine with an older, 2.6.18 kernel:

```
/lib64/libc.so.6: version `GLIBC_2.7' not found
```

The cause of the error is pretty self-explanatory - the binary needs symbols from glibc 2.7, and the target machine has an older version. The glibc wiki is [not very helpful](https://sourceware.org/glibc/wiki/FAQ#How_do_I_build_a_binary_that_works_on_older_GNU.2BAC8-Linux_distributions.3F). My biggest concern was finding out what dependency had been added - which method was I using that only existed in glibc 2.7 and up?

It turns out this is really easy to find using `objdump -T`. `objdump` is included in the `binutils` package. From the `objdump` help output:

```
-T, --dynamic-syms       Display the contents of the dynamic symbol table
```

Running `objdump -T <binary> | grep GLIBC` gives a list of all the symbols being dynamically loaded from the glibc shared object. This table was quite large, but towards the top I found:

```
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 memset
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 ftell
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 snprintf
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 abort
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 fseek
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.7   __isoc99_sscanf
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 exit
```

So `sscanf` is the method call that isn't supported. It was pretty easy in this case to rewrite the program in a way that didn't use `sscanf` - once this was done, the binary ran on the 2.6.18 kernel with no problems. 

For the future, I wrote this little bash script which finds the highest version of glibc used in a binary and compares it to a maximum allowed glibc version. This can be used as part of a build process to catch new dependencies: 

```bash
# Find the newest version of glibc required by a binary, and compare it to a fixed, maximum allowed version
BIN="$1"
MAX_ALLOWED_VER="$2"

if [ -z "$BIN" ] || [ -z "$MAX_ALLOWED_VER" ]; then
  echo "Usage: glibc_check.sh <binary> <max glibc version>"
  echo "ex. glibc_check.sh /bin/true 2.3"
  exit 1
fi

MAX_VER="$(objdump -T "$BIN" | sed -n 's/.*GLIBC_\([0-9]\.[0-9]\+\).*$/\1/p' | sort -g | tail -n 1)"

if echo "$MAX_VER $MAX_ALLOWED_VER" | awk '{exit $1>$2?0:1}'; then
	echo "FAIL - Got max GLIBC version $MAX_VER, greater than allowed max $MAX_ALLOWED_VER"
	exit 1
else
	echo "OK - Got max GLIBC version $MAX_VER, less than or equal to allowed max $MAX_ALLOWED_VER"
fi
``` 
