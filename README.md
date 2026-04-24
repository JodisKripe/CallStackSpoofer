# CallStackSpoofer

This project is a pingback to my [blog post](https://hulkops.gitbook.io/blog/red-team/x64-call-stack-spoofing)

# Do not walk through in DEBUG mode. Prologues etc get clobbered.

# How this works
- Move Current RSP to Non-Volatile Register (r15)
- Move the first argument to non volatile register (r13 )
- push 0 to demarcate the end of stack unwinding
- Create two fake frames which seem real based on how normal stacks look like in Process HAcker2 or sth
- Create a frame to trigger a JOP gadget (jmp dword ptr [ebx]) (so that when target function exits, it returns to the gadget address)
- add variables to the registers and stack for the target function
- Move the cleanup routine to a pointer withing r13 and move that address to non volatile register rbx via lea
- jump to the target function

- As the target function returns it performs a ret and the address on the stack is that of the gadget.
- The gadget jumps to the address stored in ebx which is the restore cleanup routine.
- the routine removes the fake stack frames(first push addresses and then the frame sizez) and pushes and jumps to r15 which has the original rsp pointer .

# How is frame size decided?
- Locate the exception entries -> pImgOptHdr->DataDirectory[IMAGE_DIRECTORY_ENTRY_EXCEPTION]
- Contains RUNTIME Function entries
```
typedef struct _RUNTIME_FUNCTION {
    DWORD BeginAddress;
    DWORD EndAddress;
    DWORD UnwindInfoAddress;
} RUNTIME_FUNCTION;
```
- Find the correct function's Runtime Function Entry
```
dwFuncOffset = pFuncAddr - hModule;
if (dwFuncOffset >= BeginAddress && dwFuncOffset <= EndAddress)
```
- Access it's UNWIND_INFO(`pUnwindInfo = ((PUNWIND_INFO)(hModule + pRuntimeFunction->UnwindInfoAddress))`) to understand the function prologue
```
typedef struct _UNWIND_INFO {
    UBYTE Version : 3;
    UBYTE Flags : 5;
    UBYTE SizeOfProlog;
    UBYTE CountOfUnwindCodes;
    UBYTE FrameRegister : 4;
    UBYTE FrameOffset : 4;
    UNWIND_CODE UnwindCode[...];
} UNWIND_INFO;
```
- Loop through the UNWIND_CODE entries to understand what kind of changes have to be mande to the stack to return it to a state where the function was not triggered.
```
UWOP_ALLOC_SMALL -> sub rsp, X
X is stored in UNWIND_CODE[i].OpInfo

dwStackSize += (pUnwindCode[i].OpInfo + 1) * 8;  // Value needs adjustment of + 1
```
```
UWOP_ALLOC_LARGE -> sub rsp, <large_size>
Case 1: Medium size (upto 512 KB)

Slot 0:
  UnwindOp = UWOP_ALLOC_LARGE
  OpInfo   = 0

Slot 1:
  FrameOffset = 0x28

dwStackSize += pUnwindCode[i + 1].FrameOffset * 8;
i++;

Case 2: Large size
Slot 0:
  UnwindOp = UWOP_ALLOC_LARGE
  OpInfo   = 1

Slot 1 + 2:
  Full 32-bit size value

dwStackSize += *(ULONG*)(&pUnwindCode[i + 1]);
i += 2;
```
```
UWOP_PUSH_NONVOL -> push rbx / rbp / rsi / etc
dwStackSize += 8;
if OpInfo == 4 then stacksize doesnt change
```
```
UWOP_PUSH_MACHFRAME
Case 1 → OpInfo == 0 → 40 bytes

CPU pushes:

RIP     (8 bytes)
CS      (8 bytes)
RFLAGS  (8 bytes)
RSP     (8 bytes)
SS      (8 bytes)
------------------
Total = 40 bytes
Case 2 → OpInfo == 1 → 48 bytes

Same as above plus error code:

ERROR CODE (8 bytes)
RIP
CS
RFLAGS
RSP
SS
------------------
Total = 48 bytes
```
```
UWOP_SAVE_NONVOL/UWOP_SAVE_NONVOL_FAR
mov [rsp + 0x20], rbx || mov [rsp + 0x30], rsi || mov [rsp + 0x500], rbx ; NONVOL_FAR

Basically instruction that do not affect the stack size. Only puts the values of registers on the stack.
```
