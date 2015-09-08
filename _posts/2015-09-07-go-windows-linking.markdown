---
layout: post
title:  "Golang linking on Windows"
date:   2015-09-07 00:00:00
categories: golang windows cgo
---

I ran into a strange confluence of events last week, while:

- building a Go library with C components
- on Windows (MinGW-w64)
- with Go 1.3.3

Running `go test -x .` in the work directory on Linux worked fine, but on Windows I got the following error:

```
"C:\\Go\\pkg\\tool\\windows_amd64\\6l.exe" -o "C:\\Users\\ADMINI~1\\AppData\\Local\\Temp\\go-build308712991\\_\\C_\\Users\\Administrator\\Desktop\\go-grok\\cgrok\\_test\\cgrok.test.exe" -L "C:\\Users\\ADMINI~1\\AppData\\Local\\Temp\\go-buid308712991\\_\\C_\\Users\\Administrator\\Desktop\\go-grok\\cgrok\\_test" -L "C:\\Users\\ADMINI~1\\AppData\\Local\\Temp\
go-build308712991" -w -extld=gcc "C:\\Users\\ADMINI~1\\AppData\\Local\\Temp\\go-build308712991\\_\\C_\\Users\\Administrtor\\Desktop\\go-grok\\cgrok\\_test\\main.a"
# testmain
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: getpid
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: strdup
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/269)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/231)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/231)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/56)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/202)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/231)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/202)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/139)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/113)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/231)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/84)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/231)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/139)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/170)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/354)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/354)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/354)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/354)
_/C_/Users/Administrator/Desktop/go-grok/cgrok(.text): undefined: _/C_/Users/Administrator/Desktop/go-grok/cgrok(/294)
```

The Go linker, `6l`, can't find a whole bunch of methods while linking `main.a`. For context, prior to release 1.5 Go uses the host compiler (GCC in this case) to build C into object files, and it's own compiler (`6g`) to build Go object files. It then does a final linking pass with `6l`. `6l` supports the subset of ELF (on Linux) or [PE](https://en.wikipedia.org/wiki/Portable_Executable) (on Windows) used by `6g`, but it turns out it can't link some object files produced by GCC.

Ultimately the errors were from two separate issues: 

[Issue #8811](https://github.com/golang/go/issues/8811) - `6l` doesn't support PE objects from MinGW 4.8.3 or later. This is resolved in Go 1.4. It would be possible to roll back to an old version of MinGW / GCC to get around this problem. 

[Issue #7555](https://github.com/golang/go/issues/7555) - `6l` can't find `strdup`. There's no root cause I can find, it seems like MinGW's GCC implicitly links in some extra objects that `6l` can't find. I tested Go 1.3.3, 1.4.2 and 1.5.0, and they all exhibit this issue when using internal linking.

The ultimate solution to both issues was in [issue #4069](https://github.com/golang/go/issues/4069) - Go 1.5 supports (and defaults to) external linking on Windows. This uses GCC for the final linking process, eliminating both bugs.
