# Distributing an application with embedded Python on a Mac

This document explains how to embed an isolated Python distribution within a host application on MacOS. It assumes your version number is 3.6.1 for throughout. That can be changed by replacing e.g. `3.6.1` -> `3.7.0`, `3.6` -> `3.7`, and `python36` -> `python37` throughout.


We have two main goals:

1. That users can run release builds regardless of which (if any) Python distributions exist on their system
2. That developers can build against Python headers and shared library in the project directory, so the same Xcode project works on different systems


## Gathering the Python resources

First create a directory called `python36` alongside the `.xcodeproj` file in your project's root directory. We'll copy everything we need into that directory.

Starting with version 3.5, CPython now distributes an official embeddable zip file that contains minified versions of all of its external resources. Unfortunately, they only seem to think this is useful on Windows, so all the binary resources it contains are useless on MacOS. However we can still use the collection of Python standard library bytecode they have compiled for us. Download the official embeddable zip file, and extract its zipped standard library file:

    wget https://www.python.org/ftp/python/3.6.1/python-3.6.1-embed-amd64.zip
    unzip python-3.6.1-embed-amd64.zip python36.zip -d python36
    rm python-3.6.1-embed-amd64.zip

We removed the zip file at the end, because that was all we needed from it.

The next step is to copy headers and shared library files from our system's Python installation. If you have installed Python with the CPython installer, you should find it in a directory such as:

    /Library/Frameworks/Python.framework/Versions/3.6

You might want to store that directory in a shell variable called `$pydist`.

    cp -r $pydist/Headers python36/include
    cp -r $pydist/Python python36
    cp -r $pydist/lib/python3.6/lib-dynload python36

The last line copies the contents of Python's `lib-dynload` directory. It contains shared object files for libraries like `zlib` and `cmath`. In the end we can reduce the size of this directory (from ~12 MB to less than 1) by removing libraries we don't need. But for now just copy the whole thing. Your `python36` directory should now contain the following files:

    include
    Python
    lib-dynload
    python36.zip

In order to use the Python shared library on a different system, we need to bundle it with the application. We also need to modify the shared library itself to reflect its migration from the path specific to your system, to the application bundle. Rename the shared library's `id` attribute to reflects this:

    chmod u+w python36/Python
    install_name_tool -id @executable_path/../Resources/python36/Python python36/Python

The specific directory `Resources/python36` we chose for the library's `id` attribute needs to match the `Copy Files` build phase, which we do in the next section.


## Configuring Xcode

Configuring Xcode to use our local Python resources involves four main steps:

1. Make Python headers available to compiled code
2. Link against our local Python shared library
3. Copy the Python runtime resources to the application bundle on build
4. Tell the interpreter where to find the runtime resources


For step (1) above, open Xcode, go to the 'Build Phases' and find any sources files in the 'Compile Sources' sub-section that include (directly or indirectly) `Python.h`. Add `-Ipython36/include` to the compiler flags for those sources.

Next for step (2), open a Finder window in your project directory, and drag the file `python36/Python` into the Frameworks section of the left sidebar. Symbols in the python shared library will now be resolved at build time, and your project should build. However, the application should generate a dynamic loader error if you try to run it, because application bundle's Resources directory does not contain the shared library, even though we used `install_name_tool` above to name that directory as the library's location. To get it there, we add a `Copy Files` build phase, and then add `python36/Python` to the (currently empty) list of files. Choose "Resources" as the Destination, and type `python36` as the Subpath. Now your application should build and run.

However, if Python initializes properly, that's only because it happened to locate the standard library files it needed somewhere on your system. But we want an isolated Python environment, so we need to add the runtime resources we collected to the application's bundle. Drag the `lib-dynload` directory and the `python36.zip` files from your `python36` directory into the `Copy Files` build phase, just below the Python shared library file. Now Python's standard library files are there in the bundle.

The final step is to inform Python of where it can find its runtime resources. This needs to be done just before initializing the interpreter. The Objective C++ code below locates `python36` in the bundle, sets the Python path accordingly, and then initializes the interpreter.

    NSURL* python36 = [[NSBundle mainBundle] URLForResource:@"python36" withExtension:nil];
    NSURL* dynload = [python36 URLByAppendingPathComponent:@"lib-dynload"];
    NSURL* zipfile = [python36 URLByAppendingPathComponent:@"python36.zip"];
    NSString* pythonPath = [@[dynload.path, zipfile.path] componentsJoinedByString:@":"];

    std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
    std::wstring pythonPathWide = converter.from_bytes(pythonPath.UTF8String);

    Py_SetPath(pythonPathWide.data());
    Py_Initialize();

Note that we had to do some conversion to UTF16 encoding and a wide character type. The C++ code to accomplish that requires the `locale` and `codecvt` headers to be included.
