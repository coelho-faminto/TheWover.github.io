---
layout: post
title: Emulating Covert Operations - 0: Dynamic Invocation (Avoiding PInvoke & API Hooks)
---

*TLDR: Presenting DInvoke, a new API in SharpSploit that acts as a dynamic replacement for PInvoke. Using it, we show how to dynamically invoke unmanaged code from memory or disk while avoiding API Hooking and suspicious imports.*

# Dynamic Invocation - D/Invoke

Over the past few months, myself and b33f (@FuzzySecurity, Ruben Boonen) have quitely been adding an API to SharpSploit that helps you use unmanaged code from C# while avoiding suspicious P/Invokes. Rather than statically importing API calls with PInvoke, you may use Dynamic Invocation (I call it DInvoke) to load the DLL at runtime and call the function using a pointer to its location in memory. This avoids detections that look for imports of suspicious API calls via the Import Address Table in the .NET Assembly's PE headers. Additionally, it lets you call arbitrary unmanaged code from memory (while passing parameters on the stack), allowing you to bypass API hooking in a variety of ways. Overall. DInvoke is intended to be a direct replacement for PInvoke that gives offensive tool developers great flexibility in how they can access and invoke unmanaged code. 

## Delegates

So what does DInvoke actually entail? Rather than using PInvoke to import the API calls that we want to use, we use any way we would like to load a DLL into memory. Then, we get a pointer to a function in that DLL. We may call that function from the pointer while passing in our parameters.

By leveraging this dynamic loading API rather than the static loading API that sits behind PInvoke, you avoid directly importing suspicious API calls into your .NET Assembly. Additionally, this API lets you easily invoke unmanaged code from memory in C# (passing in parameters and receiving output) without doing some hacky workaround like self-injecting shellcode.

We accomplish this through the magic of [Delegates](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/). .NET includes the Delegate API as a way of wrapping a method/function in a class. If you have ever used the Reflection API to enumerate methods in a class, the objects you were inspecting were actually a form of delegate.

The Delegate API has a number of fantastic features, such as the ability to instantiate Delegates from function pointers and to dynamically invoke the function wrapped by the delegate while passing in parameters. 

Let's take a look at how DInvoke uses these Delegates:

```csharp

/// <summary>
/// Dynamically invoke an arbitrary function from a DLL, providing its name, function prototype, and arguments.
/// </summary>
/// <author>The Wover (@TheRealWover)</author>
/// <param name="DLLName">Name of the DLL.</param>
/// <param name="FunctionName">Name of the function.</param>
/// <param name="FunctionDelegateType">Prototype for the function, represented as a Delegate object.</param>
/// <param name="Parameters">Parameters to pass to the function. Can be modified if function uses call by reference.</param>
/// <returns>Object returned by the function. Must be unmarshalled by the caller.</returns>
public static object DynamicAPIInvoke(string DLLName, string FunctionName, Type FunctionDelegateType, ref object[] Parameters)
{
    IntPtr pFunction = GetLibraryAddress(DLLName, FunctionName);
    return DynamicFunctionInvoke(pFunction, FunctionDelegateType, ref Parameters);
}

/// <summary>
/// Dynamically invokes an arbitrary function from a pointer. Useful for manually mapped modules or loading/invoking unmanaged code from memory.
/// </summary>
/// <author>The Wover (@TheRealWover)</author>
/// <param name="FunctionPointer">A pointer to the unmanaged function.</param>
/// <param name="FunctionDelegateType">Prototype for the function, represented as a Delegate object.</param>
/// <param name="Parameters">Arbitrary set of parameters to pass to the function. Can be modified if function uses call by reference.</param>
/// <returns>Object returned by the function. Must be unmarshalled by the caller.</returns>
public static object DynamicFunctionInvoke(IntPtr FunctionPointer, Type FunctionDelegateType, ref object[] Parameters)
{
    Delegate funcDelegate = Marshal.GetDelegateForFunctionPointer(FunctionPointer, FunctionDelegateType);
    return funcDelegate.DynamicInvoke(Parameters);
}

```

These two functions are the core of the DInvoke API. The second is the most important. It creates a Delegate from a function pointer and invokes the function wrapped by the delegate, passing in parameters provided by you. The parameters are passed in as an array of Objects so that you can pass in whatever data you need in whatever form. You must take care to ensure that the data passed in is structured in the way that the unmanaged code will expect.

The confusing part of this is probably the `Type FunctionDelegateType` parameter. This is where you pass in the function prototype of the unmanaged code that you want to call. If you remember from PInvoke, you set up the function with something like:

```csharp
[DllImport("kernel32.dll")]
public static extern IntPtr OpenProcess(
        ProcessAccessFlags dwDesiredAccess,
        bool bInheritHandle,
        UInt32 dwProcessId
);
```

You must also pass in a function prototype for DInvoke. This lets the Delegate know how to set up the stack when it invokes the function. If you compare this to how you would normally invoke unmanaged code from memory in C# (by self-injecting shellcode), this is MUCH easier!

Defining a delegate works in a similar manner. You can define a delegate similar to how you would define a variable. Optionally, you can specify what calling convention to use when calling the function wrapped by the delegate.

```csharp
[UnmanagedFunctionPointer(CallingConvention.StdCall)]
public delegate UInt32 NtOpenProcess(
    ref IntPtr ProcessHandle,
    Execute.Win32.Kernel32.ProcessAccessFlags DesiredAccess,
    ref Execute.Native.OBJECT_ATTRIBUTES ObjectAttributes,
    ref Execute.Native.CLIENT_ID ClientId);
```

## Using DInvoke

### Executing Code

We are building a second set of function prototypes in SharpSploit. There is already a PInvoke library; we will now build a DInvoke library in the `SharpSploit.Execution.DynamicInvoke` namespace. The DInvoke library provides a managed wrapper function for each unmanaged function. The wrapper helps the user by ensuring that parameters are passed in correctly and the correct type of object is returned.

It is worth noting: PInvoke is MUCH more forgiving about data types than DInvoke. If the data types you specify in a PInvoke function prototype are not *quite* right, it will silently correct them for you. With DInvoke, that is not the case. You must marshal data in *exactly* the correct way, ensuring that the data structures you pass in are in the same format as the unmanaged code expects. This is annoying. And is part of why we created a seperate namespace for DInvoke signatures and wrappers. If you want to understand better how to marshal data for PInvoke/DInvoke, I would recommend reading @matterpreter's [blog post on the subject](https://posts.specterops.io/offensive-p-invoke-leveraging-the-win32-api-from-managed-code-7eef4fdef16d).

The code below demonstrates how DInvoke is used for the `NtCreateThreadEx` function in `ntdll.dll`. The delegate (that sets up the function prototype) is stored in the `SharpSploit.Execution.DynamicInvoke.Native.DELEGATES` struct. The wrapper method is `SharpSploit.Execution.DynamicInvoke.Native.NtCreateThreadEx` that takes all of the same parameters that you would expect to use in a normal PInvoke.

```csharp

namespace SharpSploit.Execution.DynamicInvoke
{
    /// <summary>
    /// Contains function prototypes and wrapper functions for dynamically invoking NT API Calls.
    /// </summary>
    public class Native
    {
        public static Execute.Native.NTSTATUS NtCreateThreadEx(
            ref IntPtr threadHandle,
            Execute.Win32.WinNT.ACCESS_MASK desiredAccess,
            IntPtr objectAttributes, IntPtr processHandle,
            IntPtr startAddress,
            IntPtr parameter,
            bool createSuspended,
            int stackZeroBits,
            int sizeOfStack,
            int maximumStackSize,
            IntPtr attributeList)
        {
            // Craft an array for the arguments
            object[] funcargs =
            {
                threadHandle, desiredAccess, objectAttributes, processHandle, startAddress, parameter, createSuspended, stackZeroBits,
                sizeOfStack, maximumStackSize, attributeList
            };

            Execute.Native.NTSTATUS retValue = (Execute.Native.NTSTATUS)Generic.DynamicAPIInvoke(@"ntdll.dll", @"NtCreateThreadEx",
                typeof(DELEGATES.NtCreateThreadEx), ref funcargs);

            // Update the modified variables
            threadHandle = (IntPtr)funcargs[0];

            return retValue;
        }

```

You may use this wrapper to dynamically call `NtCreateThreadEx` as if you were calling any other managed function: 

```csharp
IntPtr threadHandle = new IntPtr();

//Dynamically invoke NtCreateThreadEx to create a thread at the address specified in the target process.
result = DynamicInvoke.Native.NtCreateThreadEx(ref threadHandle, Win32.WinNT.ACCESS_MASK.SPECIFIC_RIGHTS_ALL | Win32.WinNT.ACCESS_MASK.STANDARD_RIGHTS_ALL, IntPtr.Zero,
                    process.Handle, baseAddr, IntPtr.Zero, suspended, 0, 0, 0, IntPtr.Zero);

// The "threadHandle" variable will now be updated with a pointer to the handle for the new thread. 
```

You can, of course, manually use the Delegates in the DInvoke library without the use of our helper functions.

```csharp
 //Get a pointer to the NtCreateThreadEx function.
IntPtr pFunction = Execution.DynamicInvoke.Generic.GetLibraryAddress(@"ntdll.dll", "NtCreateThreadEx");

//Create an instance of a NtCreateThreadEx delegate from our function pointer.
DELEGATES.NtCreateThreadEx createThread = (NATIVE_DELEGATES.NtCreateThreadEx)Marshal.GetDelegateForFunctionPointer(
   pFunction, typeof(NATIVE_DELEGATES.NtCreateThreadEx));

//Invoke NtCreateThreadEx using the delegate
createThread(ref threadHandle, Win32.WinNT.ACCESS_MASK.SPECIFIC_RIGHTS_ALL | Win32.WinNT.ACCESS_MASK.STANDARD_RIGHTS_ALL, IntPtr.Zero,
                    process.Handle, baseAddr, IntPtr.Zero, suspended, 0, 0, 0, IntPtr.Zero);
```

### Calling Modules

### Loading Code
The section above shows how you would use Delegates and the DInvoke API. But how do you obtain the address for a function in the first place? The answer to that question really is: *however you would like*. But, to make that process easier, we have provided a suite of tools to help you locate and call code using a variety of mechanisms.

The easiest way to locate and execute a function is to use the `DynamicAPIInvoke` function shown above in the first code example. It uses `GetLibraryAddress` to locate a function.

* `GetLibraryAddress`: First, checks if the module is already loaded using `GetLoadedModuleAddress`. If not, it loads the module into the process using `LoadModuleFromDisk`, which uses the NT API call `LdrLoadDll` to load the DLL. Either way, it then uses `GetExportAddress` to find the function in the module. Can take a string, an ordinal number, or a hash as the identifier for the function you wish to call.
* `GetLoadedModuleAddress`: Uses `Process.GetCurrentProcess().Modules` to check if a module on disk is already loaded into the current process. If so, returns the address of that module.
* `LoadModuleFromDisk`: This will generate an Image Load ("modload") event for the process, which could be used as part of a detection signal.
* `GetExportAddress`: Starting from the base address of a module in memory, parses the PE headers of the module to locate a particular function. Can take a string, an ordinal number, or a hash as the identifier for the function you wish to call.

Additionally, we have provided several ways to load modules from memory rather than from disk.
* `MapModuleToMemory`: Manually maps a module into dynamically allocated memory, properly aligning the PE sections, correcting memory permissions, and fixing the Import Address Table. Can take either a byte array or the name of a file on disk.
* `MapModuleToMemoryAddress`: Manually maps a module that is already in memory (contained in a byte array), to a specific location in memory.
* `OverloadModule`: Uses Module Overloading to map a module into memory backed by a decoy DLL on disk. Chooses a random decoy DLL that is not already loaded, is signed, and exists in `%WINDIR%\System32`. Threads that execute code in the module will appear to be executing code from a legitimate DLL. Can take either a byte array or the name of a file on disk.

## Why?

Delegates and DInvoke presents several opportunities for offensive tool developers.

### Avoid Suspicious Imports

As previously mentioned, you can avoid statically importing suspicious API calls. If, for example, you wanted to import `MiniDumpWriteDump` from `Dbghelp.dll` you could use DInvoke to dynamically load the DLL and invoke the API call. If you were then to inspect your .NET Assembly in an Assembly dissassembler (such as dnSpy), you would find that `MiniDumpWriteDump` is not referenced in its import table.

### Manual Mapping

DInvoke supports manual mapping of PE modules, stored either on disk or in memory. This capability can be used either for bypassing API hooking or simply to load and execute payloads from memory without touching disk. The module may either be mapped into dynamically allocated memory or into memory backed by an arbitrary file on disk. When a module is manually mapped from disk, a fresh copy of it is used. That way, any hooks that AV/EDR would normally place within it will not be present. If the manually mapped module makes calls into other modules that are hooked, then AV/EDR may still trigger. But at least all calls into the manually mapped module itself will not be caught in any hooks. This is why malware often manually maps `ntdll.dll`. They use a fresh copy to bypass any hooks placed within the original copy of `ntdll.dll` loaded into the process when it was created, and force themselves to only use `Nt*` API calls located within that fresh copy of `ntdll.dll`. Since the `Nt*` API calls in `ntdll.dll` are merely wrappers for syscalls, any call into them will not inadvertantly jump into other modules that may have hooks in place. To learn more about our manual mapping, [check out our separate blog post].

A word of caution: manual mapping is complex and we do not garuantee that our implementation covers every edge case. The version we have implemented now is servicable for many common use cases and will be improved upon over time.

### Unknown Execution Flow at Compile Time

Sometimes, you may want to write a program where the flow of execution is unknown or undefined at compile time. Rather than the program being one sequential procedure, maybe it uses dynamically loaded plugins, is self-modifying, or provides an interface to the user that allows them to specify how execution should proceed. All of these are cases that would typically be considered dangerous and... also unwise life choices. But, if you write malware, then that description probably applies to the rest of your life as well. :-P DInvoke allows you to dynamically invoke arbitrary unamanaged APIs without specifying them at build-time.

### Shellcode Execution

A Delegate is effectively a wrapper for a function pointer. Shellcode is machine code that can be executed independantly. As such, if you have a pointer to it, you can execute it. SharpSploit already took advantage of delegates in order to execute shellcode in this way in the `SharpSploit.Execution.ShellCode.ShellCodeExecute` function. You could also execute shellcode using the `DynamicFunctionInvoke` method within DInvoke. Furthermore, you could use it to execute shellcode that expects parameters to be passed in on the stack or attempts to return a value.

## Example - Finding Exports

The example below demonstrates several ways of finding and calling exports of a DLL.

```csharp

///Author: b33f (@FuzzySecurity, Ruben Boonen)
using System;

namespace SpTestcase
{
    class Program
    {

        static void Main(string[] args)
        {
            // Details
            String testDetail = @"
            #=================>
            # Hello there!
            # I find things dynamically; base
            # addresses and function pointers.
            #=================>
            ";
            Console.WriteLine(testDetail);

            // Get NTDLL base form the PEB
            Console.WriteLine("[?] Resolve Ntdll base from the PEB..");
            IntPtr hNtdll = SharpSploit.Execution.DynamicInvoke.Generic.GetPebLdrModuleEntry("ntdll.dll");
            Console.WriteLine("[>] Ntdll base address : " + string.Format("{0:X}", hNtdll.ToInt64()) + "\n");

            // Search function by name
            Console.WriteLine("[?] Resolve function by walking the export table in-memory..");
            Console.WriteLine("[+] Search by name --> NtCommitComplete");
            IntPtr pNtCommitComplete = SharpSploit.Execution.DynamicInvoke.Generic.GetLibraryAddress("ntdll.dll", "NtCommitComplete", true);
            Console.WriteLine("[>] pNtCommitComplete : " + string.Format("{0:X}", pNtCommitComplete.ToInt64()) + "\n");

            Console.WriteLine("[+] Search by ordinal --> 0x260 (NtSetSystemTime)");
            IntPtr pNtSetSystemTime = SharpSploit.Execution.DynamicInvoke.Generic.GetLibraryAddress("ntdll.dll", 0x260, true);
            Console.WriteLine("[>] pNtSetSystemTime : " + string.Format("{0:X}", pNtSetSystemTime.ToInt64()) + "\n");

            Console.WriteLine("[+] Search by keyed hash --> 138F2374EC295F225BD918F7D8058316 (RtlAdjustPrivilege)");
            Console.WriteLine("[>] Hash : HMACMD5(Key).ComputeHash(FunctionName)");
            String fHash = SharpSploit.Execution.DynamicInvoke.Generic.GetAPIHash("RtlAdjustPrivilege", 0xaabb1122);
            IntPtr pRtlAdjustPrivilege = SharpSploit.Execution.DynamicInvoke.Generic.GetLibraryAddress("ntdll.dll", fHash, 0xaabb1122);
            Console.WriteLine("[>] pRtlAdjustPrivilege : " + string.Format("{0:X}", pRtlAdjustPrivilege.ToInt64()) + "\n");

            // Pause execution
            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();
        }
    }
}

```

Let's walk through the example in sequence:

1) Get the base address of `ntdll.dll`. It is loaded into every Windows process when it is initialized, so we know that it will already be loaded. As such, we can safely search the PEB's list of loaded modules to find a reference to it. Once we've found its base address from the PEB, we print the address.
2) Use `GetLibraryAddress` to find an export within `ntdll.dll` by name.
3) Use `GetLibraryAddress` to find an export within `ntdll.dll` by ordinal.
4) Use `GetLibraryAddress` to find an export within `ntdll.dll` by keyed hash.

*TODO: Extend this example to include Manual mapping and GetExportAddress*

[4_Resolve.png]

### Example - Syscall Execution

The following example

```csharp

using System;
using System.Runtime.InteropServices;

namespace SpTestcase
{
    class Program
    {
        [Flags]
        public enum ProcessAccessFlags : uint
        {
            All = 0x001F0FFF,
            Terminate = 0x00000001,
            CreateThread = 0x00000002,
            VirtualMemoryOperation = 0x00000008,
            VirtualMemoryRead = 0x00000010,
            VirtualMemoryWrite = 0x00000020,
            DuplicateHandle = 0x00000040,
            CreateProcess = 0x000000080,
            SetQuota = 0x00000100,
            SetInformation = 0x00000200,
            QueryInformation = 0x00000400,
            QueryLimitedInformation = 0x00001000,
            Synchronize = 0x00100000
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct OBJECT_ATTRIBUTES
        {
            public ulong Length;
            public IntPtr RootDirectory;
            public IntPtr ObjectName;
            public ulong Attributes;
            public IntPtr SecurityDescriptor;
            public IntPtr SecurityQualityOfService;
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct CLIENT_ID
        {
            public IntPtr UniqueProcess;
            public IntPtr UniqueThread;
        }

        [UnmanagedFunctionPointer(CallingConvention.StdCall)]
        public delegate UInt32 NtOpenProcess(
            ref IntPtr hProcess,
            ProcessAccessFlags processAccess,
            ref OBJECT_ATTRIBUTES objAttribute,
            ref CLIENT_ID clientid);

        static void Main(string[] args)
        {
            // Details
            String testDetail = @"
            #=================>
            # Hello there!
            # I dynamically generate a Syscall stub
            # for NtOpenProcess and then open a
            # handle to a PID.
            #=================>
            ";
            Console.WriteLine(testDetail);

            // Read PID from args
            Console.WriteLine("[?] PID: " + args[0]);

            // Create params for Syscall
            IntPtr hProc = IntPtr.Zero;
            OBJECT_ATTRIBUTES oa = new OBJECT_ATTRIBUTES();
            CLIENT_ID ci = new CLIENT_ID();
            Int32 ProcID = 0;
            if (!Int32.TryParse(args[0], out ProcID))
            {
                return;
            }
            ci.UniqueProcess = (IntPtr)(ProcID);

            // Generate syscall stub
            Console.WriteLine("[+] Generating NtOpenProcess syscall stub..");
            IntPtr pSysCall = SharpSploit.Execution.DynamicInvoke.Generic.GetSyscallStub("NtOpenProcess");
            Console.WriteLine("[>] pSysCall    : " + String.Format("{0:X}", (pSysCall).ToInt64()));

            // Use delegate on pSysCall
            NtOpenProcess fSyscallNtOpenProcess = (NtOpenProcess)Marshal.GetDelegateForFunctionPointer(pSysCall, typeof(NtOpenProcess));
            UInt32 CallRes = fSyscallNtOpenProcess(ref hProc, ProcessAccessFlags.All, ref oa, ref ci);
            Console.WriteLine("[?] NtStatus    : " + String.Format("{0:X}", CallRes));
            if (CallRes == 0) // STATUS_SUCCESS
            {
                Console.WriteLine("[>] Proc Handle : " + String.Format("{0:X}", (hProc).ToInt64()));
            }

            Console.WriteLine("[*] Pausing execution..");
            Console.ReadLine();
        }
    }
}


```

[3_Syscall.png]

## Room for Improvement

# Conclusions

Next up, an in-depth exploration of how to leverage SharpSploit to execute PE modules from memory, either for post-exploitation or for hook evasion. 