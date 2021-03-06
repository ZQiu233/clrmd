# Getting Started

This tutorial introduces you to the concepts of working with
`Microsoft.Diagnostics.Runtime.dll` (called 'ClrMD' for short), and the
underlying reasons why we do things the way we do. If you are already familiar
with the dac private API, you should skip down below to the code which shows you
how to create a `ClrRuntime` instance from a crash dump and a dac.

## CLR Debugging, a brief introduction

All of .NET debugging support is implemented on top of a dll we call "The Dac".
This file (usually named `mscordacwks.dll`) is the building block for both our
public debugging API (`ICorDebug`) as well as the two private debugging APIs: The
SOS-Dac API and IXCLR.

In a perfect world, everyone would use `ICorDebug`, our public debugging API.
However a vast majority of features needed by tool developers such as yourself
is lacking from `ICorDebug`. This is a problem that we are fixing where we can,
but these improvements go into CLR v.next, not older versions of CLR. In fact,
the `ICorDebug` API only added support for crash dump debugging in CLR v4.
Anyone debugging CLR v2 crash dumps cannot use `ICorDebug` at all!

The other two debugging APIs suffer from a different problem. They can be used
on a crash dump, but these APIs are considered private. Any tool which builds on
top of these APIs cannot be shipped publicly. This is due to Microsoft policy
that we only ship tools based on public APIs (unless you are the team which owns
the private API, thus CLR can ship SOS without a problem, since CLR owns both
the private SOS-Dac API and SOS itself).

The second problem with the two private debugging APIs is that they are
incredibly difficult to use correctly. Writing correct code on top of the SOS-Dac
API requires you to basically understand the entirety of the CLR's internals.
For example, how a GC Segment is laid out, how object references inside an
object can be found, and so on.

ClrMD is an attempt to bridge the gap between private APIs and tool writers. The
API itself is built on top of the Dac private API, but abstracts away all of the
gory details you are required to know to use them successfully. Better yet,
ClrMD is a publicly shipping, documented API. You may build your own programs
on top of it and ship them outside of Microsoft (which is not true if you build
on top of the raw Dac private APIs).

## What do I need to debug a crash dump with ClrMD?

As mentioned before, all .NET debugging is implemented on top of the Dac. To
debug a crash dump or live process, all you need to do is have the crash dump
and matching `mscordacwks.dll`. Those are the only prerequisites for using this
API.

The correct dac for a particular crash dump can be obtained through a
simple symbol server request. See the later Getting the Dac from the Symbol
Server section below for how to do this.

There is one other caveat for using ClrMD though: ClrMD must load and use the
dac to do its work. Since the dac is a native DLL, your program is tied to the
architecture of the dac. This means if you are debugging an x86 crash dump, the
program calling into ClrMD must be running as an x86 process. Similarly for an
amd64 crash dump. This means you will have to relaunch your tool under wow64 (or
vice versa) if you detect that the dump you are debugging is not the same
architecture as you currently are.

## Loading a crash dump

To get started the first thing you need to do is to create a `DataTarget`. The
`DataTarget` class represents a crash dump or live process you want to debug.
To create an instance of the `DataTarget` class, call one of the static functions
on `DataTarget`.  Here is the code to create a `DataTarget` from a crash dump:

```cs
using (DataTarget dataTarget = DataTarget.LoadCrashDump(@"c:\work\crash.dmp"))
{
}
```

The `DataTarget` class has two primary functions: getting information about what
runtimes are loaded into the process and creating `ClrRuntime` instances.

To enumerate the versions of CLR loaded into the target process, use
`DataTarget.ClrVersions`:

```cs
foreach (ClrInfo version in dataTarget.ClrVersions)
{
    Console.WriteLine("Found CLR Version: " + version.Version);

    // This is the data needed to request the dac from the symbol server:
    ModuleInfo dacInfo = version.DacInfo;
    Console.WriteLine("Filesize:  {0:X}", dacInfo.FileSize);
    Console.WriteLine("Timestamp: {0:X}", dacInfo.TimeStamp);
    Console.WriteLine("Dac File:  {0}", dacInfo.FileName);

    // If we just happen to have the correct dac file installed on the machine,
    // the "LocalMatchingDac" property will return its location on disk:
    string dacLocation = version.LocalMatchingDac;
    if (!string.IsNullOrEmpty(dacLocation))
        Console.WriteLine("Local dac location: " + dacLocation);

    // You may also download the dac from the symbol server, which is covered
    // in a later section of this tutorial.
}
```

Note that `target.ClrVersions` is an `IList<ClrInfo>`. We can have two copies of
CLR loaded into the process in the side-by-side scenario (that is, both v2 and
v4 loaded into the process at the same time).  `ClrInfo` also has information 
about the version of the dac you need to debug this process.  In practice though,
you should (hopefully) not need to manually download the dac.

The next step to getting useful information out of ClrMD is to construct an
instance of the `ClrRuntime` class.  This class represents one CLR runtime
in the process.  To create one of these classes, use `ClrInfo.CreateRuntime`
and you will create the runtime for the selected version:

```cs
ClrInfo runtimeInfo = dataTarget.ClrVersions[0];  // just using the first runtime
ClrRuntime runtime = runtimeInfo.CreateRuntime();
```

You can also create a runtime from a dac location on disk if you know exactly
where it is:

```cs
ClrInfo runtimeInfo = dataTarget.ClrVersions[0];  // just using the first runtime
ClrRuntime runtime = runtimeInfo.CreateRuntime(@"C:\work\mscordacwks.dll");
```

Lastly, note that `CreateRuntime` with no parameters is equivalent to checking
`ClrInfo.LocalMatchingDac`, and if that is null, ClrMD will attempt to download
the correct dac from the symbol server using `DataTarget.SymbolLocator.FindBinary`.

We will cover what to actually do with a `ClrRuntime` object in the next few
tutorials.

## Getting the Dac from the Symbol Server

When you call CreateRuntime without specifying the location of mscordacwks.dll,
ClrMD attempts to locate the dac for you.  It does this through a few mechanisms,
first it checks to see if you have the same version of CLR that you are attempting
to debug on your local machine.  If so, it loads the dac from there.  (This is usually
at c:\windows\Framework[64]\[version]\mscordacwks.dll.)  If you are debugging a crash
dump that came from another computer, you will have to find the dac that matches the
crash dump you are debugging.

All versions of the dac are requried to be on the Microsoft public symbol server,
located here:  https://msdl.microsoft.com/download/symbols.  The DataTarget.SymbolLocator
property is how ClrMD interacts with symbol servers.  If you have set the _NT_SYMBOL_PATH
environment variable, ClrMD will use that string as your symbol path.  If this
environment variable is not set, it will default to the Microsoft Symbol Server.

With any luck, you should never have to manually locate the dac or interact with
DataTarget.SymbolLocator.  CreateRuntime should be able to successfully locate
all released builds of CLR.

However, if you have built .Net Core yourself from source or are using a non-standard
build, you will have to keep track of the correct dac yourself (these will not be
on the symbol servers).  In that case you will need to pass the path of the dac
on disk to ClrInfo.CreateRuntime manually.

## Attaching to a live process

CLRMD can also attach to a live process (not just work from a crashdump). To do
this, everything is the same, except you call `DataTarget.AttachToProcess`
instead of `DataTarget.LoadCrashDump`. For example:

```cs
DataTarget dataTarget = DataTarget.AttachToProcess(0x123, AttachFlags.Noninvasive, 5000);
```

The parameters to the function are: the pid to attach to, the type of debugger
attach to use, and a timeout to use for the attach.

There are three different `AttachFlags` which can be used when attaching to a
live process: Invasive, Noninvasive, and Passive. An Invasive attach is a normal
debugger attach. Only one debugger can attach to a process at a time. This means
that if you are already attached with Visual Studio or an instance of Windbg, an
invasive attach through ClrMD will fail. A non-invasive attach gets around this
problem. Any number of non-invasive debuggers may be attached to a process. Both
an invasive and non-invasive attach will pause the debugee (so this means VS's
debugger will not function again until you detach). The primary difference
between the two is that you cannot control the target process or receive debug
notifications (such as exceptions) when using a non-invasive attach.

To be clear though, the difference between an invasive and non-invasive attach
doesn't matter to CLRMD. It only matters if you need to control the process
through the `IDebug` interfaces. If you do not care about getting debugger events
or breaking/continuing the process, you should choose a non-invasive attach.

One last note on invasive and non-invasive is that managed debuggers (such as
`ICorDebug`, and Visual Studio) cannot function when something pauses the
process. So if you attach to a process with a Noninvasive or Invasive attach,
Visual Studio's debugger will hang until you detach.

A "Passive" attach does not involve the debugger apis at all, and it does not
pause the process. This means things like Visual Studio's debugger will continue
to function. However, if the process is running, you will get highly
inconsistent information for things like heap data, as the target process is
continuing to run as you attempt to read data from it.

In general, you should use a Passive attach if you are using CLRMD in
conjunction with another debugger (like Visual Studio), and only when that
debugger has the target process paused. You should use an invasive attach if you
need to control the target process. You should use a non-invasive attach for all
other uses of the API.

## Detaching from a process or dump

`DataTarget` implements the `IDisposable` interface. Every instance of
`DataTarget` should be wrapped with a `using` statement (otherwise you should find
a place to call `Dispose` when you are done using the `DataTarget` in your
application).

This is important for two reasons. First, any crash dump you load will be locked
until you dispose of `DataTarget`. Second, and more importantly is the live
process case. For a live process, ClrMD acts as a real debugger, which has a lot
of implications in terms of program termination. Primarily, if you kill the
debugger process without detaching from the target process, Windows will kill
the target process. Calling `DataTarget`'s `Dispose` method will detach from
any live process.

`DataTarget` itself has a finalizer (which calls `Dispose`), and this will be run
if the process is terminated normally. However, I highly recommend that your
program eagerly calls `Dispose` as soon as you are done using ClrMD on the
process.

The next tutorial will cover some basic uses of the `ClrRuntime` class.

Next Tutorial: [The ClrRuntime Object](ClrRuntime.md)
