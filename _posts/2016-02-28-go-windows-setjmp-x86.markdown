---
layout: post
title:  "setjmp/longjmp on 64-bit MinGW"
date:   2016-02-29 00:00:00
categories: golang windows cgo 64-bit setjmp longjmp
---

Although it seems pretty gnarly, using `setjmp` and `longjmp` to implement exceptions in C is apparently a time-honoured tradition. For those who aren't familiar, calling `setjmp` populates a `jmp_buf` structure with the current program state - register contents, PC, etc. Calling `longjmp` on  that `jmp_buf` will return to the line where `setjmp` was called, with the same state - provided you haven't returned from the function where `setjmp` was called already. `longjmp` unwinds the stack until it finds the right frame, allowing you to return from multiple layers of function calls without having to actually return error codes. An example:

```

int someFunc() {
	jmp_buf jmp;
	int err;
	// Call setjmp - returns 0 the first time around
	if (err = setjmp(jmp)) {
		// We're back here after an exception occurred
		return err;
	} else {
		someOtherFunc(jmp);
	}
}

void someOtherFunc(jmp_buf jmp) {
	someOtherOtherFunc(jmp);
}

void someOtherOtherFunc(jmp_buf jmp) {
	// Throw an exception - go back to someFunc and return 1 as the error code
	// This avoids having to handle error codes in someOtherFunc
	longjmp(jmp, 1);
}

```

I ran into a situation where some legacy C code I was integrating into a Go codebase used `setjmp`/`longjmp` on 64-bit Windows. Unfortunately, all I got when an exception occurred was:

```
Exception 0xc0000028 0x1300000 0x32000032 0x77537e68
PC=0x77537e68
signal arrived during external code execution
```

And then a stack trace. This wasn't super helpful, but since the program only crashed when an exception occurred, I figured `longjmp` was causing the problem. There are a couple Github tickets which suggested Go's internal linker broke `longjmp`:

- [Issue 9754](https://github.com/golang/go/issues/9754)
- [Issue 13672](https://github.com/golang/go/issues/13672)

But I was using Go 1.5, so the default was external linking! This looked like it might actually be a problem with MinGW-w64. 

I tried running the binaries in a debugger, which showed that the error was happening in `msvcrt.dll`, but not much else. Windows debugging is not much fun.  There's [a mailing list thread](https://sourceforge.net/p/mingw-w64/bugs/406/) which seemed to suggest that `longjmp` was broken for other people in MinGW-w64. 

Following the thread's advice, I was able to work around the issue by using the compiler's built-in functions `__builtin_setjmp` and `__builtin_longjmp` instead of `setjmp` and `longjmp`. These seem to be alternative implementations included in GCC which don't call `msvcrt.dll`. The downside is that the `__builtin` functions are GCC-specific and not technically user-facing. They could change in any release. But using the built-in functions did fix the crash bug, and now exception handling is working correctly in my project.
