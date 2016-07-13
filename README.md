# PythonForWindows

PythonForWindows is a base of code aimed to make interaction with `Windows` (on X86/X64) easier (for both 32 and 64 bits Python).
Its goal is to offer abstractions around some of the OS features in a (I hope) pythonic way.
It also tries to make the barrier between python and native execution thinner in both ways.
There is no external dependencies but it relies heavily on the `ctypes` module.


Some of this code is clean (IMHO) and some parts are just a wreck that works for now.
Let's say that the codebase evolves with my needs and my curiosity.

Complete online documentation is available [here][ONLINE_DOC]
You can find some examples of code in the [samples directory][SAMPLE_DIR] or [online][ONLINE_SAMPLE].

Parts of PythonForWindows are used in the [LKD project][LKD_GITHUB].

## Overview

### Processes / Threads

PythonForWindows offers objects around processes and allows you to:

- Retrieve basic process informations (pid, name, ppid, bitness, ...)
- Perform basic interprocess operation (allocation, create thread, read/write memory)
- Explore the PEB (Process Environment Block)
- Execute `native` and `Python` code in the context of a process.

I try my best to make those features available for every cross-bitness processes (`32 <-> 64` in both ways).
This involves relying on non-documented `Windows` functions/behaviours and also injecting code in the 64bits world of a `Syswow64` process.
All those operations are also available for the `current_process`.

You can also make some operation on threads (suspend/resume/wait/get(or set) context/ kill)

```python
>>> import windows
>>> windows.current_process.bitness
32
>>> calc = [p for p in windows.system.processes if p.name == "calc.exe"][0]
>>> calc
<WinProcess "calc.exe" pid 6960 at 0x37391f0>
>>> calc.bitness
64
>>> calc.peb.modules[:3]
[<RemoteLoadedModule64 "calc.exe" at 0x3671e90>, <RemoteLoadedModule64 "ntdll.dll" at 0x3671030>, <RemoteLoadedModule64 "kernel32.dll" at 0x3671080>]
>>> k32 = calc.peb.modules[2]
>>> hex(k32.pe.exports["CreateFileW"])
'0x7ffee6761550L'
>>> calc.threads[0]
<WinThread 3932 owner "calc.exe" at 0x3646350>
>>> hex(calc.threads[0].context.Rip)
'0x7ffee68b54b0L'
>>> calc.execute_python("import os")
True
>>> calc.execute_python("exit(os.getpid() + 1)")
# execute_python raise if process died
Traceback (most recent call last):
...
ValueError: Unknown exit code  0xc000004bL
>>> calc
<WinProcess "calc.exe" pid 6960 (DEAD) at 0x37391f0>
>>> calc.exit_code
6961L
```


### IAT Hook

This codebase is born from my need to have IAT hooks implemented in Python.
So the features is present (See [online documentation][ONLINE_IATHOOK] about IAT hooks).


### Winproxy

A wrapper around some Windows functions. Arguments name and order are the same,
but some have default values and the functions raise exception on call error (I don't like 'if' around all my call).


### Native execution

To make the barrier between `native` and `Python` code,
PythonForWindows allows you to create native function callable from Python (thanks `ctypes`) and also embed
a simple x86/x64 assembler.

```python
>>> import windows.native_exec.simple_x86 as x86
>>> code = x86.MultipleInstr()
>>> code += x86.Mov("EAX", 41)
>>> code += x86.Inc("EAX")
>>> code += x86.Ret()
>>> code.get_code()
'\xc7\xc0)\x00\x00\x00@\xc3'
# Create a function that takes no parameters and return an uint
>>> f = windows.native_exec.create_function(code.get_code(), [ctypes.c_uint])
>>> f()
42L
```

### Wintrust

To easily script some signature check script, PythonForWindows implements some wrapper functions around ``wintrust.dll``

```python
>>> import windows.wintrust
>>> windows.wintrust.is_signed(r"C:\Windows\system32\ntdll.dll")
True
>>> windows.wintrust.is_signed(r"C:\Windows\system32\python27.dll")
False
>>> windows.wintrust.full_signature_information(r"C:\Windows\system32\ntdll.dll")
SignatureData(signed=True,
    catalog=u'C:\\Windows\\system32\\CatRoot\\{F750E6C3-38EE-11D1-85E5-00C04FC295EE}\\Package_35_for_KB3128650~31bf3856ad364e35~amd64~~6.3.1.2.cat',
    catalogsigned=True, additionalinfo=0L)
>>> windows.wintrust.full_signature_information(r"C:\Windows\system32\python27.dll")
SignatureData(signed=False, catalog=None, catalogsigned=False, additionalinfo=TRUST_E_NOSIGNATURE(0x800b0100L))
```

### WMI

To extract/play with even more information about the system, PythonForWindows is able to perform WMI request.

```python
>>> import windows
>>> windows.system.wmi.select
<bound method WmiRequester.select of <windows.winobject.wmi.WmiRequester object at 0x036BA590>>
>>> windows.system.wmi.select("Win32_Process", ["Name", "Handle"])[:4]
[{'Handle': u'0', 'Name': u'System Idle Process'}, {'Handle': u'4', 'Name': u'System'}, {'Handle': u'412', 'Name': u'smss.exe'}, {'Handle': u'528', 'Name': u'csrss.exe'}]
# Get WMI data for current process
>>> wmi_cp = [p for p in windows.system.wmi.select("Win32_Process") if int(p["Handle"]) == windows.current_process.pid][0]
>>> wmi_cp["CommandLine"], wmi_cp["HandleCount"]
(u'"C:\\Python27\\python.exe"', 227)
```

### Registry

The project also contains some wrapping classes around `_winreg` for simpler use.

```python
>>> import windows
>>> from windows.generated_def import KEY_WRITE, KEY_READ, REG_QWORD
>>> registry = windows.system.registry
>>> cuuser_software = registry(r'HKEY_CURRENT_USER\Software')
>>> cuuser_software
<PyHKey "HKEY_CURRENT_USER\Software">
>>> cuuser_software.sam
KEY_READ(0x20019L)
# Explore subkeys
>>> cuuser_software.subkeys[:3]
[<PyHKey "HKEY_CURRENT_USER\Software\7-Zip">, <PyHKey "HKEY_CURRENT_USER\Software\AppDataLow">, <PyHKey "HKEY_CURRENT_USER\Software\Audacity">]
>>> tstkey = registry('HKEY_CURRENT_USER\TestKey',  KEY_WRITE | KEY_READ)
# Get / Set individual value
>>> tstkey["VALUE"] = 'a_value_for_my_key'
>>> tstkey["VALUE"]
KeyValue(name='VALUE', value=u'a_value_for_my_key', type=1)
>>> tstkey["MYQWORD"] = (123456789987654321, REG_QWORD)  # Default is REG_DWORD for int/long
>>> tstkey["MYQWORD"]
KeyValue(name='MYQWORD', value=123456789987654321L, type=11)
# Explore Values
>>> tstkey.values
[KeyValue(name='MYQWORD', value=123456789987654321L, type=11), KeyValue(name='VALUE', value=u'a_value_for_my_key', type=1)]
```

### Debugger

PythonForWindows provides a standard debugger to debug other processes.

```python
import windows
import windows.debug
import windows.test
import windows.native_exec.simple_x86 as x86

from windows.test import pop_calc_32
from windows.generated_def import EXCEPTION_ACCESS_VIOLATION

class MyDebugger(windows.debug.Debugger):
    def on_exception(self, exception):
        code = exception.ExceptionRecord.ExceptionCode
        addr = exception.ExceptionRecord.ExceptionAddress
        print("Got exception {0} at 0x{1:x}".format(code, addr))
        if code == EXCEPTION_ACCESS_VIOLATION:
            print("Access Violation: kill target process")
            self.current_process.exit()

calc = windows.test.pop_calc_32(dwCreationFlags=DEBUG_PROCESS)
d = MyDebugger(calc)
calc.execute(x86.assemble("int3; mov [0x42424242], EAX; ret"))
d.loop()

## Ouput ##
Got exception EXCEPTION_BREAKPOINT(0x80000003L) at 0x77e13c7d
Got exception EXCEPTION_BREAKPOINT(0x80000003L) at 0x230000
Got exception EXCEPTION_ACCESS_VIOLATION(0xc0000005L) at 0x230001
Access Violation: kill target process
```

The debugger handles

* Standard breakpoint ``int3``
* Hardware Execution breakpoint ``DrX``
* Memory breakpoint ``virtual protect``


#### LocalDebugger

You can also debug your own process (or debug a process by injection) via the LocalDebugger.

The LocalDebugger is an abstraction around Vectored Exception Handler (VEH)

```python
import windows
from windows.generated_def.winstructs import *
import windows.native_exec.simple_x86 as x86

class SingleSteppingDebugger(windows.debug.LocalDebugger):
    SINGLE_STEP_COUNT = 4
    def on_exception(self, exc):
        code = self.get_exception_code()
        context = self.get_exception_context()
        print("EXCEPTION !!!! Got a {0} at 0x{1:x}".format(code, context.pc))
        self.SINGLE_STEP_COUNT -= 1
        if self.SINGLE_STEP_COUNT:
            return self.single_step()
        return EXCEPTION_CONTINUE_EXECUTION

class RewriteBreakpoint(windows.debug.HXBreakpoint):
    def trigger(self, dbg, exc):
        context = dbg.get_exception_context()
        print("GOT AN HXBP at 0x{0:x}".format(context.pc))
        # Rewrite the infinite loop with 2 nop
        windows.current_process.write_memory(self.addr, "\x90\x90")
        # Ask for a single stepping
        return dbg.single_step()


d = SingleSteppingDebugger()
# Infinite loop + nop + ret
code = x86.assemble("label :begin; jmp :begin; nop; ret")
func = windows.native_exec.create_function(code, [PVOID])
print("Code addr = 0x{0:x}".format(func.code_addr))
# Create a thread that will infinite loop
t = windows.current_process.create_thread(func.code_addr, 0)
# Add a breakpoint on the infitine loop
d.add_bp(RewriteBreakpoint(func.code_addr))
t.wait()
print("Done!")

## Output ##

Code addr = 0x6a0002
GOT AN HXBP at 0x6a0002
EXCEPTION !!!! Got a EXCEPTION_SINGLE_STEP(0x80000004L) at 0x6a0003
EXCEPTION !!!! Got a EXCEPTION_SINGLE_STEP(0x80000004L) at 0x6a0004
EXCEPTION !!!! Got a EXCEPTION_SINGLE_STEP(0x80000004L) at 0x6a0005
EXCEPTION !!!! Got a EXCEPTION_SINGLE_STEP(0x80000004L) at 0x770c7c04
Done!

```

The local debugger handles

* Standard breakpoint ``int3``
* Hardware Execution breakpoint ``DrX``

### Other stuff (see doc / samples)

- Network
- Services
- COM


[LKD_GITHUB]: https://github.com/sogeti-esec-lab/LKD/
[SAMPLE_DIR]: https://github.com/hakril/PythonForWindows/tree/master/samples
[ONLINE_DOC]: http://hakril.github.io/PythonForWindows/
[ONLINE_SAMPLE]: http://hakril.github.io/PythonForWindows/sample.html
[ONLINE_IATHOOK]: http://hakril.github.io/PythonForWindows/iat_hook.html