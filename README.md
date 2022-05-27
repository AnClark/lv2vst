LV2 - VST wrapper
=================

Expose LV2 plugins as VST2 plugins to a VST plugin-host on Windows, OSX and Linux.


QuickStart
----------

On GNU/Linux:
```bash
git clone https://github.com/x42/lv2vst
cd lv2vst
make
mkdir -p ~/.vst/lv2vst
cp lv2vst.so ~/.vst/lv2vst/lv2vst.so

# this assume you have x42-plugins, specifically x42-eq installed
# or use `lv2ls` to find some LV2 URI/URI-prefixes.
echo "http://gareus.org/oss/lv2/fil4#mono" > ~/.vst/lv2vst/.whitelist
echo "http://gareus.org/oss/lv2/fil4#stereo" >> ~/.vst/lv2vst/.whitelist
```

Then launch a LinuxVST plugin host...


Description
-----------

lv2vst can be deployed in different variants:

*  wrap dedicated LV2 bundle(s), compile-time specified.
*  wrap any/all LV2 Plugins(s), runtime lookup

The first is intended for plugin-authors: A LV2 plugin can be
seamlessly distributed as VST2.

The second approach is useful for users who want to use
existing LV2s in a VST plugin-host.

Specifically:

1. If a .bundle file exists in the same dir as the VST,
   only load lv2 bundle(s) specified in the file
   (dirs relative to lv2vst.dll, one per line).
   A list of bundles can alternatively be specified at compile-time and
   hard-coded.

*otherwise* use system-wide LV2 world:

2. Load .whitelist and .blacklist files if they exist
   in the same dir as the VST (one URI per line).

   If the whitelist file exists and contains a single complete URI
   (one line), no VST-shell is used and a single plugin is exposed.

   Otherwise index all plugins (alike `lv2ls`), expose only plugins
   which URI matches a prefix in the whitelist.
   If no whitelist files is present (or if it's empty), consider all
   system-wide LV2 plugins.

   Next the .blacklist file is tested, if a Plugin-URI matches a prefix
   specified in the blacklist, it is skipped.

The CRC32 of the LV2 URI is used as VST-ID.

LV2VST does not bridge CPU architectures. The LV2 plugins and LV2VST
architectures and ABIs need to match.

A dedicated bundle (1), or dedicated whitelist (2) is preferred over
blacklisting.

Supported LV2 Features
----------------------

* LV2:ui, native UI only: X11UI on Linux, CocoaUI on OSX and WindowsUI on Windows.
* LV2 Atom ports (currently at most only 1 atom in, 1 atom output)
    * MIDI I/O.
    * plugin to GUI communication
    * LV2 Time/Position
* LV2 URI map
* LV2 Worker thread extension
* LV2 State extension
* Latency reporting (port property)


Build and Install via GNU Make
-----------------

Compiling lv2vst requires gnu-make and the GNU c/c++-compiler. 

Windows (.dll) versions are to be cross-compiled on GNU/Linux using MinGW. I (AnClark) suggest you to build on Msys2. 

- Build on Linux:

```bash
  make
```

- Build on Windows (via Msys2). Run `mingw64.exe` or `mingw32.exe` in your Msys2's install path to setup build environment.

```bash
  # Install build essentials
  pacman -Sy mingw-w64-x86_64-gcc make      # 64-bit
  pacman -Sy mingw-w64-i686-gcc make        # 32-bit

  # Clean and build
  make XWIN=x86_64-w64-mingw32 clean all    # 64-bit
  make XWIN=i686-w64-mingw32 clean all      # 32-bit
```

You will get either `lv2vst.so` or `lv2vst.dll`. Copy it into your host's search path. On Windows, `lv2vst.dll` is statically-linked, no extra files (especially `libwinpthread-1.dll`) needed.

For macOS/OSX, a .vst bundle folder needs to be created, with the plugin in
Contents/MacOS/, see `make osxbundle`.

lv2vst can be used multiple-times in dedicated folders, each with specific
whitelist to expose plugin-collections. e.g.

```bash
	mkdir -p ~/.vst/plugin-A/
	mkdir -p ~/.vst/plugin-B/
	cp lv2vst.so ~/.vst/plugin-A/
	cp lv2vst.so ~/.vst/plugin-B/
	lv2ls | grep URI-A-prefix > ~/.vst/plugin-A/.whitelist
	lv2ls | grep URI-B-prefix > ~/.vst/plugin-B/.whitelist
```

## Build and Install via CMake

I (AnClark) also CMake-ized the original Makefile. Using CMake can be more reliable on different platform, and can be much faster so you can have a better experience on debugging.

Run these commands to configure and build:

```bash
cmake -S . -B build
cmake --build build
```

Output files are called `liblv2vst.so` or `liblv2vst.dll`.

**NOTICE:**

- CMake build is not implemented on macOS, as I can't afford a MacBook Pro :cry:
- Built DLL file will link against `libwinpthread-1.dll`, so it's not suitable for distribution. If you want to distribute your own Win32 build, try "Build and Install via GNU Make" above.

Caveats
-------

Various LV2 plugins are known to cause crashes.

LV2VST first indexes all plugins to be collected into a single VST-shell.
This step is generally safe (using only lv2 ttl meta-data). The LV2 description
is validated and VST-IDs (CRC32 of the LV2-URI) are generated.

When the host scans the actual plugins, lv2vst instantiates the mapped LV2.
This is to ensure that the actual plugin will load later.
A single VST-shell.so will load (and unload/destroy) various LV2 plugins
(and their dependent libs, if any) in the same memory-space.

This is known to cause problems with some sub-standard LV2 plugins, e.g.
some use static init, fini methods, others use custom threads which are not
terminated (or the plugin does not wait for threads to terminate: crash on unload).
Yet others use static globals or expose all their symbols in the global namespace
(possibly conflicting names) and/or mix libaries which do so.

Most of these plugins will also cause crashes in other LV2 hosts. Except that LV2
hosts don't generally instantiate a plugin unless it is used.

Hence it is preferred to expose only specific plugins that are known to work
reliably, using a whitelist.

YMMV
