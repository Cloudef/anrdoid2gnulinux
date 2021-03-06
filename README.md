# android2gnulinux

AOSP/Bionic compatiblity layer for GLIBC systems

## Notes

Currently heavy work in progress. Linker will be rewritten as it's
utter crap. Need to add logging/tracing facility controlled from env.
That's about it, otherwise it's just implementing stuff. `libjvm-*.so`
implementations are not correct at all, but the point is to not make
correct implementations yet, just to get some games run, which are pretty
forgiving for incorrect implementation :D.

## Building

```shell
# to compile for android ABI x86 <https://developer.android.com/ndk/guides/abis>
make x86
```

Other than x86 is currently untested and probably doesn't work.

## Running

```shell
./andre my-android-binary
./andre my-android-jni-lib.so
sh andre-apk.sh my-android-unity-app.apk
```

`andre-apk.sh` currently assumes app that uses `libunity.so`.

## Notes about runtime libraries and porting

AOSP/bionic libs in `runtime` (the non `libjvm-*.so` ones) can be used to
port android software to GNU/linux also. The runtime libraries contain
`bionic_` prefixed symbols for runtime compatibility only. Functions that
are not implemented in glibc, but are in AOSP/bionic are implemented
normally in these libraries. This means that the same libraries can be
used for both, compiling and linking to GNU/linux android native code, and
running AOSP/bionic linked code. The PORTING and RUNTIME capability is not
optional compile-time, nor do we separate PORTING and RUNTIME libraries.
This is for pure simplicity.

If you have to load/use bionic linked libraries from your ported
application, you can use the `libdl.so` that has `bionic_` prefixed
versions of the usual `dlfcn.h` functions.

If for some reason one wants minimal library for either PORTING or RUNTIME
task, here are the rules you can apply for symbol removal:

```
PORTING: Remove all symbols with prefix `bionic_`. Don't do this if you need `bionic_dlopen` from `runtime/libdl.so`.
RUNTIME: Remove the symbol if there also exists the same symbol with a `bionic_` prefix, retaining only the `bionic_` edition.
```

That said, the libraries are pretty small anyways.

## JNI and JVM implementations

We implement the JNI interface and simulate a rather unusual JVM that
only calls C functions. Native android code that uses JNI, thinks
it's talking to java but in reality it's calling functions from the
`runtime/libjvm-*.so` libraries. The `runtime/libjvm-*.so` libraries
implement java apis referenced by the application in C.

In future, there might be very small dalvik bytecode interpreter, that we
can use to eliminate the need of porting internal java dependencies and
implementing entrypoint for every native application. We would only have
to implement the external java apis that exist in android system image in
C, and yes, this also means you could run android java applications.

If you are porting android application to GNU/Linux that heavily relies
on JNI, you can use the `libjvm.so` to use the minimal JVM inside the
application, without having to rewrite the JNI reliant code. At this
moment however, you still have to manually do the initial JNI calls
that setup and initialize your application. This may not be needed
eventually, if we implemented what was described above (minimal dalvik
bytecode interpreter).

## Related Work

[apkenv](https://github.com/thp/apkenv.git) This is what inspired this
project. The current linker code originates from this project (modified
from AOSP). apkenv's in pretty messy state though and doesn't do clean
implementation. It's still interesting approach and you should check it
out. Also helpful resource for this project.

[libhybris](https://github.com/libhybris/libhybris) Very badly documented
project, but I guess it has some overlap with this project? Seems to be
mainly aimed for using android GPU drivers on GNU/Linux. Also uses AOSP
linker, probably this is the resource apkenv used.

## Contributing

Currently it's not yet contribution friendly. When time comes, I'll write
more detailed topic on this. Project also needs some regression testing
for all the functions that will be implemented. As for implementing stuff,
people should only implement the stuff some application needs. Lets not
implement anything that doesn't get used.

## Debugging with apitrace

Since apitrace and similiar OpenGL debuggers redirect the dlopen calls,
you have to `LD_PRELOAD` the `runtime/libEGL.so` while running apitrace.
`runtime/libEGL.so` is responsible for wrapping `eglGetDisplay` and
`eglCreateWindowSurface` functions, so that valid native handles are
passed to the real system EGL implementation.

```
LD_PRELOAD=runtime/libEGL.so apitrace -a egl ./app bioniclib.so
```

## Q & A

### Why? I still don't get what this is for...

Basically, most android games are just native libraries with a very small
java entrypoint that does some boilerplate app setup, and few calls to the
native library. Native library may communicate back to java through JNI,
but usually the java parts are very small in these kinds of applications.
Thus these kind of applications will work fine with even barebones JNI/JVM
implementation which doesn't neccessary even have to be very "correct" :).
To use the native libraries on GNU systems though we need to use custom
linker as bionic's linker is incompatible with glibc. Similarly bionic
libc is incompatible with glibc and we have to handle these differences to
be able to use android libraries on GNU/linux. We also have to implement
the system libraries that exists on android, but not on traditional
GNU/Linux, such as the stuff in AOSP that handle window/context creation,
input and such. These implementations can be valuable for porters, or even
application developers as some interfaces may be very clean.

### Why not use dalvikvm?

While we could use dalvikvm to handle the JNI and java, we still would
have to reimplement all the android java apis to be GNU/Linux compatible,
or run android services in container or something. This is probably
what `anbox` and such projects do. The scope of this project is however
something that does not need anything from android system. The minimal
bytecode interpreter would be fun research project to see how little
code we can achieve all this with. Of course java's standard library
is quite large so the work for reimplementing java apis should not be
underestimated.

Our minimal JNI/JVM implementation is clearly separated from other
AOSP/bionic components however, so it's perfectly possible for someone
to use dalvikvm and android java libraries/services for the java bits
instead. If someone's gonna do and maintain that work and it doesn't
feel too clunky/hacky/invasive I have nothing against such code in this
project. (aka, the JNI/JVM implementation is not hardcoded)

### Wait so this also implements java libraries in C, can we use these implementations from java?

This is interesting and weird question at the same time. But yes, we
could generate java classes that just specifies each method as native
and thus calls our C implementation. However at current scope these
implementations are not very useful. The goal is to just provide some
barebones to get the interesting android applications to work. Correctness
is not guaranteed! This project may be useful for NDK development however,
basically anything where you don't use java much nor need a very "correct
java implementation" :).

### How about arm libraries?

You can use qemu. For hw acceleration, we could use custom libGLES libs
that communicate with host system. Though such libraries may exist
already, maybe even inside qemu project.. I haven't researched.
