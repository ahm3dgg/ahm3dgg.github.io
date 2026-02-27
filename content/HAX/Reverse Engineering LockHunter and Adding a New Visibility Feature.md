### Introduction

According To LockHunter's official website, LockHunter is 

> It is a free tool to delete files blocked by something you do not know. LockHunter is useful for fighting against malware, and other programs that are blocking files without a reason. Unlike other similar tools it deletes files into the recycle bin so you may restore them if deleted by mistake.

Its also written in Delphi.

I use this tool a lot, and I really like it, however across my use I noticed that it was missing something, sometimes the file was locked but the tool showed that no process is locking the file, until I was testing some code that had to deal with Section Objects on Windows, So before going into reverse engineering, Lets go through some theory on Section Objects on Windows.
### Introduction To Windows Section Objects

Section Objects are basically Windows way of implementing shared memory and memory mapped files, we have two types of sections 

- Page-File Backed Sections
- File Backed Sections

Page-File Backed Sections are basically shared memory buffers, that processes can you use to communicate between each either as a method of doing Interprocess-Communication, they are named Page-File Backed, because when its time for the working set manager to free up some physical memory, if the page is part of a page-file backed section and is dirty meaning that it was modified, its content will be written to the page file on the system, which is typically located at `C:\Windows\pagefile.sys`

File Backed Sections are memory mapped files, they can be data files or executables, when its an executable we call it an image backed section, they are called File Backed because they refer to an actual file on disk something like `travis_scoot_top_hits.txt` or `don_toliver_scareware.exe`, for memory mapped files that are data files, when they are dirty or have been modified and was mapped as read/write, the content is reflected to the file on disk, however when they are mapped as as read-only writes won't reflect on the file on disk.

For Executable files, they are mapped as copy-on-write and data written to memory is not reflect on disk, in that case if a process writes to a page that has copy-on-write on it, it will receive a private copy of that page, so it doesn't disrupt the view for other processes running and sharing that executable image, that's also the reason when hooking `ntdll.dll` or `kernel32.dll`, or any executable that is shared, your executable is shared as well, the hook gets applied to your process virtual address space only, unless you explicitly wrote to a target process.

Also when you create an Image Section, the file gets locked.

Let's look at an example on how to works with Sections using the NT API.

```cpp
#include <phnt_windows.h>
#include <phnt.h>
#include <cstdio>

#pragma comment(lib, "ntdll.lib")

int wmain(int argc, wchar_t* argv[])
{
	HANDLE FileHandle{};
	HANDLE Section{};
	NTSTATUS Status{};
	OBJECT_ATTRIBUTES ObjectAttributes{};
	IO_STATUS_BLOCK IoStatus{};
	UNICODE_STRING NtFilePath{};
	wchar_t* DosFilePath{};

	if (argc < 2)
	{
		printf("Pass a file path\n");
		return 1;
	}

	DosFilePath = argv[1];
	RtlDosPathNameToNtPathName_U(DosFilePath, &NtFilePath, nullptr, nullptr);

	InitializeObjectAttributes(&ObjectAttributes, &NtFilePath, OBJ_CASE_INSENSITIVE, nullptr, nullptr);
	Status = NtOpenFile(
		&FileHandle, 
		SYNCHRONIZE | FILE_READ_DATA, 
		&ObjectAttributes,
		&IoStatus, 
		FILE_SHARE_READ|FILE_SHARE_WRITE, 
		FILE_SYNCHRONOUS_IO_NONALERT
	);

	if (!NT_SUCCESS(Status))
	{
		printf("Failed to open File Handle | Status -> %lX\n", Status);
		return 1; 
	}

	printf("Opened File Handle -> %p\n", FileHandle);
	
	Status = NtCreateSection(
		&Section, 
		SECTION_ALL_ACCESS, 
		nullptr, 
		nullptr, 
		PAGE_READONLY, 
		SEC_IMAGE,
		FileHandle
	);

	if (!NT_SUCCESS(Status))
	{
		printf("Failed to create a section | Status -> %lX\n", Status);
		return 1;
	}

	PVOID BaseAddress = NULL;
	SIZE_T ViewSize = PAGE_SIZE;
	Status = NtMapViewOfSection(Section,
		ZwCurrentProcess(),
		&BaseAddress,
		0,
		0,
		NULL,
		&ViewSize,
		ViewUnmap,
		0,
		PAGE_READONLY
	);

	printf("Mapped File at -> %p\n", BaseAddress);

	printf("Close File Handle\n");
	getchar();
	NtClose(FileHandle);

	printf("Close Section Handle ?\n");
	NtClose(Section);
	getchar();

	printf("Unmap View of Section ?\n");
	getchar();
	NtUnmapViewOfSection(NtCurrentProcess(), BaseAddress);
}
```

I won't really go through every single argument I pass here you can reference `ntdoc` for that, however what I am doing is here is basically

- Open a Handle To an Executable File using `NtOpenFile`

- Creating a Section Object using `NtCreateSection` passing `SEC_IMAGE` to indicate that this an Executable-file backed section

- Mapping the sections using `NtMapViewOfSection` to make it available in my process virtual address space. 

- I also introduced some `getchar()`s, just to see what will happen when I close each one at a time, that will help us when trying to understand LockHunter's behaviour

if I ran this program I named `Sections.exe` against some test executable I named `test.exe`, and then launch LockHunter on that file, we should see

![[Pasted image 20260226112117.png]]

Here we can see that the tool is indicating that `Sections.exe` is locking `test.exe`, its also shows one instance of `test.exe`, what LockHunter is trying to tell here, that `Sections.exe` is holding One Handle to the file named `test.exe`

if you tried renaming or deleting that file you won't be able to, or opening it for instance in 010 editor you will see this lock on the file

![[Pasted image 20260226113347.png]]

So maybe let's try to close the file handle, and see if that works, also let's run LockHunter on the file again, when we do saw we are faced with the green checkmark surely the file is unlocked and we can do whatever 

![[Pasted image 20260226113501.png]]

Okay now I am able to rename and copy the file, let's try deleting it

![[Pasted image 20260226113638.png]]

Oh oh I can't, but explorer tells us the name of the process that is using the file, so its a nice thing.

okay let's see if I can write to the file by opening it in 010 editor, well I am still faced with the lock on the file (its not the same image trust me I am not lying to you)

![[Pasted image 20260226113826.png]]

In the example code above, also closing the section handle won't solve our problem because we mapped the section, if we haven't mapped the section, closing the section handle will solve the problem, or if there is some buggy program that unmapped the section and forgot to close the section handle, closing the section handle will work here, at this point I will probably just terminate the program or unmap the section, tools like SystemInformer gives us more visibility into this, 

![[Pasted image 20260226114257.png]]

So what about bringing that visibility into LockHunter ? maybe from a practical perspective its not worth the effort and I could just use SystemInformer but I really wanted to use this project as a learning exercise for me, it allowed me to get more into windows kernel development and reverse engineering drivers.

In the following subsections, I will be describing how I added visibility of section handles into LockHunter.
### LockHunter Bird's Eye View Reverse Engineering

LockHunter is written in Delphi, as we can see from Detect It Easy Tool.

![[Pasted image 20260227084949.png]]

I also think looking at the LockHunter's folder would be a good point to start, since we can start identifying the files it uses.

![[Pasted image 20260226115505.png]]

We can here see the main executable among other files, some DLLs for the shell extension for the context menu, and what caught my attention is the driver `USRFindHandle64.sys`, well Its name says that its finding a handle, at this point we don't really know what's really doing.

We can also see some other files like `LHService.exe` this is a service that is used to facilitate deleting a file after next reboot.

We certainly have to start somewhere, perhaps let's try to answer some concrete questions

- How Does LockHunter Queries the System Handles ?
- How Does LockHunter Closes a Handle ?

### How Does LockHunter Queries System Handles ?

To answer this question, and especially when reversing engineering a not relatively small program like this, its helpful to flip roles and think like a forward engineer here, the question really becomes how can we query system handles on windows ?

if you googled that question, you will soon find about [NtQuerySystemInformation](https://ntdoc.m417z.com/ntquerysysteminformation), which has the following definition which I grabbed from `ntdoc` 

```cpp
NTSYSCALLAPI
NTSTATUS
NTAPI
NtQuerySystemInformation(
    _In_ SYSTEM_INFORMATION_CLASS SystemInformationClass,
    _Out_writes_bytes_opt_(SystemInformationLength) PVOID SystemInformation,
    _In_ ULONG SystemInformationLength,
    _Out_opt_ PULONG ReturnLength
    );
```

this is an Native API, that actually does what it says *queries system global information*, this function works by providing it a System Information Class, this basically a number that tells the function what kind of information you want to query, one of such information classes is `SystemHandleInformation`, this information class allows us to query all system handles, literally all of them even handles that are part of the `System` process, the data returned has the following structure

```cpp
typedef struct _SYSTEM_HANDLE_INFORMATION
{
    ULONG NumberOfHandles;
    _Field_size_(NumberOfHandles) SYSTEM_HANDLE_TABLE_ENTRY_INFO Handles[1];
} SYSTEM_HANDLE_INFORMATION, *PSYSTEM_HANDLE_INFORMATION;

// Note: This information class is deprecated since values are limited to 65535. Use SystemExtendedHandleInformation instead.
typedef struct _SYSTEM_HANDLE_TABLE_ENTRY_INFO
{
    USHORT UniqueProcessId;
    USHORT CreatorBackTraceIndex;
    UCHAR ObjectTypeIndex;
    UCHAR HandleAttributes;
    USHORT HandleValue;
    PVOID Object;
    ACCESS_MASK GrantedAccess;
} SYSTEM_HANDLE_TABLE_ENTRY_INFO, *PSYSTEM_HANDLE_TABLE_ENTRY_INFO;
```

Few interesting things in this structure, that matters the most for us

- UniqueProcessId: The Process Owning that handle
- ObjectTypeIndex: The ID of the Object that the handle refers to (File, Section, Registry Key, ...)
- HandleValue: The Actual Handle Value
- Object: Kernel Address of the Object the handle is referring to

It's important to note that though the field `Object` holds the kernel address of the object, usermode programs calling this function, won't be able to read or write to that address, this is because while the kernel is mapped into every process for performance reasons such as avoiding page table swaps, and TLB flushes, the page table entries of the kernel are marked as supervisor and are only accessible from kernel mode or cpu supervisor mode, though usermode programs can read the address and send it to a kernel driver running in kernel mode that can do read/writes on their behave.

As you can also see, ntdoc warns about this information class, and says its deprecated and we should use `SystemExtendedHandleInformation` this is because the type of `HandleValue` is of size 16-bits which is only limited 65535 possible values, `SystemExtendedHandleInformation` raises the limit to be 64-bit on 64-bit systems and 32-bit on 32-bit systems.

So Now we have some information, maybe we can fire up IDA Pro, and start looking if there are any calls to this function, I started by examining the Import Address Table, and Yes I found it

![[Pasted image 20260226122944.png]]

In usermode `Nt` and `Zw` are equivalent, `ZwQuerySystemInformation` will map to `NtQuerySystemInformation` in NTDLL's export table.

`ZwQuerySystemInformation` is called from one function, which I have given the name `QuerySystemInformation` at `0x0640EE0`

```cpp
PVOID __fastcall QuerySystemInformation(SYSTEM_INFORMATION_CLASS InformationClass)
{
  ULONG ReturnLength; // [rsp+20h] [rbp+20h] BYREF
  NTSTATUS i; // [rsp+24h] [rbp+24h]
  PVOID SystemInformation; // [rsp+28h] [rbp+28h] BYREF
  ULONG SystemInformationLength; // [rsp+34h] [rbp+34h]
  __int64 vars38; // [rsp+38h] [rbp+38h]

  vars38 = 0;
  SystemInformationLength = 0xFFFF;
  SystemInformation = Alloc(0xFFFF);
  for ( i = ZwQuerySystemInformation(InformationClass, SystemInformation, 0xFFFFu, &ReturnLength);
		i == STATUS_INFO_LENGTH_MISMATCH;
		i = ZwQuerySystemInformation(InformationClass, SystemInformation, SystemInformationLength, &ReturnLength) )
  {
	SystemInformationLength *= 2;
	ReAlloc(&SystemInformation, SystemInformationLength);
  }
  if ( !i )
	return SystemInformation;
  sub_405AB0(SystemInformation);
  return vars38;
}
```

To understand what its doing, we have to understand how `ZwQuerySystemInformation` works, this function works by providing it a pointer to the buffer that it will fill up with the information requested, and the size of that buffer, and in case the buffer is not large enough a `STATUS_INFO_LENGTH_MISMATCH` error is returned, so the programmer will usually try to allocate in a loop until a `STATUS_SUCCESS` is returned, here its doubling the size every time it the function fails.

Doing XREFs on `QuerySystemInformation`, we can see its called from a function The Programmer named `ScanForLockingHandles` at `0x6412D0`.

I mean I love it when programmers leave their logging messages :D

![[Pasted image 20260226123842.png]]

![[Pasted image 20260226123931.png]]

Let's focus on the relevant parts of this function, the first relevant thing the function does is obtaining the file object type index, each windows executive object has some id that identifies it, for example on Windows 10 Files has ID 37 and sections have ID 42, this ID however changes between different versions of windows.

To accomplish this, and not hardcode IDs, since they can change, 
LockHunter here does a neat trick in order to obtain the File Object Type Index

```cpp
NULFileHandle = CreateFileW_0(L"NUL", 0x80000000, 0, 0, 3u, 0, 0);

if ( NULFileHandle == INVALID_HANDLE_VALUE )
{
	ReportWin32Error();
}

SystemInformation = QuerySystemInformation(SystemHandleInformation);

if ( !SystemInformation )
{
	ReportWin32Error();
}

CurrentProcessId = GetCurrentProcessId_0();
HandleCount = SystemInformation->NumberOfHandles - 1;
i = 0;

if ( HandleCount >= 0 )
{
	++HandleCount;
	while ( *&SystemInformation->Handles[i].UniqueProcessId != CurrentProcessId || SystemInformation->Handles[i].HandleValue != NULFileHandle )
	{
		if ( ++i == HandleCount )
		{
			goto LABEL_16;
		}
	}
	FileObjectTypeIndex = SystemInformation->Handles[i].ObjectTypeIndex;
}
```

Here the function, opens a handle on the windows `NUL` device, which is similar to `/dev/null` on Linux, if you are familiar with that, it then queries all system handles, and loops until it reaches the handle entry pointing to the `NUL` device, and since this `NUL` device is an object of type `File`, we can then use its `ObjectTypeIndex` field, which will be the same for all File Handles, Sounds cool right ?

The Next part of the code, will query the system handle information again, IDK Why to be honest, they could have used the previous information received, regardless, the function will filter out handles that are part of our process and handles that don't refer to files, it then calls to a function I named `QueryFileObjectInfo` at `0x0643DB0` which we will discuss below, here is how IDA Thinks the code should look like with some minor modifications

```cpp
  ProcessInformation = QuerySystemInformation(SystemProcessInformation);
  if ( ProcessInformation )
  {
    SystemInformation = QuerySystemInformation(SystemHandleInformation);
    if ( SystemInformation )
    {
      *(a1 + 0x70) = 0;
      *(a1 + 0x78) = SystemInformation->NumberOfHandles;
      *(a1 + 0x74) = 1;
      *(a1 + 0x98) = SystemInformation->NumberOfHandles;
      *(a1 + 0x94) = 0;
      v8 = SystemInformation->NumberOfHandles - 1;
      i = 0;
      if ( v8 >= 0 )
      {
        v9 = v8 + 1;
        while ( 1 )
        {
          if ( SystemInformation->Handles[i].ObjectTypeIndex != FileObjectTypeIndex
            || *&SystemInformation->Handles[i].UniqueProcessId == hProcess.dwProcessId )
          {
            goto NextHandleEntry;
          }
          ++*(a1 + 0x94);
          if ( *(a1 + 0x15) )
          {
            sub_6419E0(0, vars58);
            sub_641A10(0, vars58);
            sub_641A40(0, vars58);
            sub_641A60(0, vars58);
            sub_641B00(0, vars58);
            goto LABEL_45;
          }
          v10 = sub_640DC0();
          if ( !QueryFileObjectInfo(v10, &SystemInformation->Handles[i], &vars78, &vars80) )
            goto NextHandleEntry;
```

Understanding `QueryFileObjectInfo` requires looking at the driver, however in brief the function takes in a pointer to the handle entry retrieved by `NtQuerySystemInformation`, and reads the kernel object address from the `Object` field, it then copies it in the IOCT input buffer, it then sends an IOCTL to the kernel driver `USRFindHandle64.sys`

```cpp
__int64 __fastcall QueryFileObjectInfo(
        __int64 LockHunterCtx,
        PSYSTEM_HANDLE_TABLE_ENTRY_INFO pHandleEntry,
        __int64 DiskVolumePath,
        __int64 FilePath)
{
  __int64 v4; // rax
  BOOL Result; // eax
  DWORD BytesReturned; // [rsp+44h] [rbp+44h] BYREF
  bool vars48; // [rsp+48h] [rbp+48h]
  _HUNTER_FIND_FILENAME_RESPONSE OutBuffer; // [rsp+49h] [rbp+49h] BYREF
  PVOID InBuffer; // [rsp+48Fh] [rbp+48Fh] BYREF
  struct _SYSTEM_HANDLE_TABLE_ENTRY_INFO HandleEntry; // [rsp+498h] [rbp+498h]

  HandleEntry = *pHandleEntry;
  if ( *(LockHunterCtx + 0x28) != 1 )
  {
    LOBYTE(pHandleEntry) = 1;
    v4 = sub_43A3A0(off_426AB8, pHandleEntry, L"The driver should be initialized before its using");
    sub_40AF20(v4);
  }
  BytesReturned = 0;
  SetMemory(&OutBuffer.FilePathPresent, 0x446, 0);
  InBuffer = HandleEntry.Object;
  Result = DeviceIoControl(*(LockHunterCtx + 8), dwIoControlCode, &InBuffer, 8u, &OutBuffer, 0x446u, &BytesReturned, 0);
  vars48 = Result;
  if ( Result )
  {
    if ( OutBuffer.FilePathPresent || OutBuffer.DiskVolumePresent )
    {
      CopyString(DiskVolumePath, OutBuffer.DiskVolumePath);
      CopyString(FilePath, OutBuffer.FilePath);
      return 1;
    }
    else
    {
      return 0;
    }
  }
  else
  {
    sub_5EF730(L"Cannot convert handle to file name, using driver");
    return 0;
  }
}
```

Its was not entirely clear for me statically, what was the IOCTL code, so I hooked my x64dbg and placed a breakpoint on `DeviceIoControl`

![[Pasted image 20260226134821.png]]

[DeviceIoControl](https://learn.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-deviceiocontrol) has the following definition 

```cpp
BOOL DeviceIoControl(
  [in]                HANDLE       hDevice,
  [in]                DWORD        dwIoControlCode,
  [in, optional]      LPVOID       lpInBuffer,
  [in]                DWORD        nInBufferSize,
  [out, optional]     LPVOID       lpOutBuffer,
  [in]                DWORD        nOutBufferSize,
  [out, optional]     LPDWORD      lpBytesReturned,
  [in, out, optional] LPOVERLAPPED lpOverlapped
);
```

Following the x64 calling convention, `dwIoControlCode` should be at rdx, which in this case contains the value `0x9C402400`, we will look at how IOCTLs are encoded in the next section.

At this point we have the following information 

- It uses `NtQuerySystemInformation` to query all system handles
- It only cares about File Handles
- It sends an IOCTL with Input Buffer containing a File Object Kernel Address
### Analyzing USRFindHandle64.sys

The Driver is actually very small, it starts at `DriverEntry`

```cpp
NTSTATUS __stdcall DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
  PDRIVER_CONTEXT DeviceExtension; // rcx
  PDEVICE_OBJECT DriverContext; // [rsp+28h] [rbp-20h] BYREF
  NTSTATUS Status; // [rsp+30h] [rbp-18h]

  DriverObject->MajorFunction[IRP_MJ_CREATE] = IrpHandler;
  DriverObject->MajorFunction[IRP_MJ_CLOSE] = IrpHandler;
  DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = IrpHandler;
  DriverObject->DriverUnload = DriverUnload;
  Status = CreateDevice(L"\\Device\\USR_Find_Handle0", 0x9C40u, DriverObject, &DriverContext);
  if ( Status >= 0 )
  {
    DeviceExtension = DriverContext->DeviceExtension;
    DeviceExtension->DeviceObject = DriverContext;
    DeviceExtension->DeviceType = 0x9C40;
  }
  return Status;
}
```

DriverEntry will set the the IRP Handlers, in the DriverObject Dispatch Table, so when the usermode application calls into the driver, the kernel knows which functions to call, it also registers an unload routine, and create a device object, here we can see this path `\\Device\\USR_Find_Handle0`, this is the path that will be used by the usermode application, the usermode application typically will open a file handle to the device object, using this path, and then pass that handle to `DeviceIoControl` or `NtDeviceIoControlFile`

`IrpHandler` is the most important function here, so let's take a look at it, I have cleaned the decompilation for it 

```cpp
NTSTATUS __stdcall IrpHandler(PDEVICE_OBJECT DeviceObject, IRP *Irp)
{
  PIO_STACK_LOCATION IoStackLocation; // [rsp+20h] [rbp-28h]
  NTSTATUS Status; // [rsp+28h] [rbp-20h]
  PVOID DeviceContext; // [rsp+30h] [rbp-18h]
  UCHAR MajorFunction; // [rsp+38h] [rbp-10h]
  ULONG IoControlCode; // [rsp+3Ch] [rbp-Ch]

  Irp->IoStatus.Information = 0;
  DeviceContext = DeviceObject->DeviceExtension;
  IoStackLocation = Irp->Tail.Overlay.CurrentStackLocation;
  Status = STATUS_NOT_IMPLEMENTED;
  MajorFunction = IoStackLocation->MajorFunction;
  if ( !IoStackLocation->MajorFunction || MajorFunction == IRP_MJ_CLOSE )
  {
    Status = STATUS_SUCCESS;
  }
  else if ( MajorFunction == IRP_MJ_DEVICE_CONTROL )
  {
    IoControlCode = IoStackLocation->Parameters.DeviceIoControl.IoControlCode;
    switch ( IoControlCode )
    {
      case IOCTL_QUERY_FILE_OBJECT_INFO:
        Status = UsrQueryFileObjectInfo(DeviceContext, Irp, IoStackLocation);
        break;
      case IOCTL_WRITE_SOME_U32_VALUE:
        Status = UsrWritesSomeU32Value(DeviceContext, Irp, IoStackLocation);
        break;
      case IOCTL_CLOSE_HANDLE_IN_CURRENT_PROCESS:
        Status = UsrCloseHandle(DeviceContext, Irp, IoStackLocation);
        break;
    }
  }
  Irp->IoStatus.Status = Status;
  IofCompleteRequest(Irp, 0);
  return Status;
}
```

I have given names to the IOCTLs

```cpp
enum USR_IOCTLS
{
	IOCTL_QUERY_FILE_OBJECT_INFO = 0x9C402400,
	IOCTL_WRITE_SOME_U32_VALUE = 0x9C402404,
	IOCTL_CLOSE_HANDLE_IN_CURRENT_PROCESS = 0x9C402408,
};
```

We have seen that the usermode application, sends an IOCTL with code `0x9C402400`, I haven't seen the others being called also I don't really know what is the purpose of the second one, since again I haven't seen them getting called.

Now Let's look at `UsrQueryFileObjectInfo`, since this what the usermode application is interested in, here is a clean decompilation of it

```cpp
NTSTATUS __stdcall UsrQueryFileObjectInfo(PVOID DeviceContext, PIRP Irp, PIO_STACK_LOCATION IoStackLocation)
{
  PHUNTER_FIND_FILENAME_RESPONSE SystemBuffer; // [rsp+30h] [rbp-58h]
  PDEVICE_OBJECT Object; // [rsp+40h] [rbp-48h]
  PFILE_OBJECT FileObject; // [rsp+58h] [rbp-30h]
  POBJECT_NAME_INFORMATION ObjectNameInfo; // [rsp+60h] [rbp-28h]
  ULONG ReturnLength; // [rsp+68h] [rbp-20h] BYREF
  unsigned int Length; // [rsp+6Ch] [rbp-1Ch]
  unsigned int ObjectNameLength; // [rsp+70h] [rbp-18h]

  SystemBuffer = (PHUNTER_FIND_FILENAME_RESPONSE)Irp->AssociatedIrp.SystemBuffer;
  if ( IoStackLocation->Parameters.DeviceIoControl.InputBufferLength != 8
    || IoStackLocation->Parameters.Read.Length != sizeof(_HUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE) )
  {
    return STATUS_INVALID_PARAMETER;
  }
  FileObject = *(PFILE_OBJECT *)Irp->AssociatedIrp.SystemBuffer;
  SystemBuffer->FileNamePresent = 0;
  SystemBuffer->DiskVolumePresent = 0;
  if ( FileObject->Type != 5 )
    return STATUS_INVALID_PARAMETER;
  SystemBuffer->Type = FileObject->Type;
  SystemBuffer->Size = FileObject->Size;
  SystemBuffer->DeviceObject = FileObject->DeviceObject;
  SystemBuffer->LockOperation = FileObject->LockOperation;
  SystemBuffer->DeletePending = FileObject->DeletePending;
  SystemBuffer->ReadAccess = FileObject->ReadAccess;
  SystemBuffer->WriteAccess = FileObject->WriteAccess;
  SystemBuffer->DeleteAccess = FileObject->DeleteAccess;
  SystemBuffer->SharedRead = FileObject->SharedRead;
  SystemBuffer->SharedWrite = FileObject->SharedWrite;
  SystemBuffer->SharedDelete = FileObject->SharedDelete;
  SystemBuffer->Flags = FileObject->Flags;
  SystemBuffer->CurrentByteOffset = FileObject->CurrentByteOffset.QuadPart;
  SystemBuffer->Waiters = FileObject->Waiters;
  SystemBuffer->Busy = FileObject->Busy;
  if ( FileObject->FileName.Length <= 0x200u )
    Length = FileObject->FileName.Length;
  else
    Length = 0x200;
  memset(SystemBuffer->FileName, 0, sizeof(SystemBuffer->FileName));
  memmove(SystemBuffer->FileName, FileObject->FileName.Buffer, Length);
  SystemBuffer->FileNamePresent = 1;
  Object = FileObject->DeviceObject;
  if ( Object->Type == 3 )
  {
    SystemBuffer->DeviceObject_Type = Object->Type;
    SystemBuffer->DeviceObject_Size = Object->Size;
    SystemBuffer->DeviceObject_RefCount = Object->ReferenceCount;
    SystemBuffer->DriverObject = Object->DriverObject;
    SystemBuffer->DeviceObject_Flags = Object->Flags;
    SystemBuffer->DeviceObject_Characteristics = Object->Characteristics;
    SystemBuffer->DeviceObject_DeviceType = Object->DeviceType;
    memset(SystemBuffer->DiskVolumePath, 0, sizeof(SystemBuffer->DiskVolumePath));
    ObjectNameInfo = (POBJECT_NAME_INFORMATION)ExAllocatePool(PagedPool, 0x200u);
    if ( ObjectNameInfo )
    {
      ObjectNameInfo->Name.MaximumLength = 0x1F8;
      if ( ObQueryNameString(Object, ObjectNameInfo, 0xFCu, &ReturnLength) >= 0 )
      {
        if ( ObjectNameInfo->Name.Length <= 512u )
          ObjectNameLength = ObjectNameInfo->Name.Length;
        else
          ObjectNameLength = 512;
        memmove(SystemBuffer->DiskVolumePath, ObjectNameInfo->Name.Buffer, ObjectNameLength);
      }
      ExFreePoolWithTag(ObjectNameInfo, 0);
    }
    SystemBuffer->DiskVolumePresent = 1;
  }
  Irp->IoStatus.Information = sizeof(_HUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE);
  return 0;
}
```

Before digging into the function, let's take some time to understand how IOCTLs are encoded, we can use a tool like [Zezula IOCTL Decoder](http://www.zezula.net/en/tools/ioctl.html)

![[Pasted image 20260226140533.png]]

Here is a description of what you are looking at 

- Device Type: This is the device type you set when calling `IoCreateDevice`, one driver can create multiple device objects, for instance a network driver like `tcpip.sys` can create multiple device objects each for each protocol `TCP`, `UDP`, `IP`, `RawIP`.

- Function: Identifies the function or action to take, in this example it asking to query file object info

- Method: Identifies how the usermode application and the driver will perform I/O, in this case we have `METHOD_BUFFERED`, which is called Buffered I/O, which in case of writes, the kernel will copy the user input buffer into a system buffer allocated from a non-paged pool, and in case of writes the kernel copies the system buffer into the usermode buffer, the same system buffer is used for reads/writes, and its located at `Irp->AssociatedIrp.SystemBuffer`, and its size is maximum between input buffer and output buffer passed in `NtDeviceIoControlFile`, other Methods can be found on MSDN

- Access: Indicates the type of access that a caller must request when opening the file object that represents the device, the possible values are `FILE_ANY_ACCESS`, `FILE_READ_DATA` and `FILE_WRITE_DATA`

So now going back to `UsrQueryFileObjectInfo` function, since we now know that its using Buffered I/O and we also know from reversing the usermode application that it sends the kernel object address of a File object, we can starting changing types in IDA, so the decompilation looks cleaner, for example we can see its reading a pointer from the system buffer, which as I said above its located at `Irp->Associated.SystemBuffer`, I have set this to PFILE_OBJECT. 

```cpp
FileObject = *(PFILE_OBJECT*)*Irp->AssociatedIrp.SystemBuffer;
```

after doing that, the output will look much cleaner, I think the field names are self explanatory, and we can also start forming the data structure that will be returned to the user

```cpp
typedef struct _HUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE
{
	BOOLEAN FileNamePresent;
	UINT16 Type;
	UINT16 Size;
	PVOID DeviceObject;
	BOOLEAN LockOperation;
	BOOLEAN DeletePending;
	BOOLEAN ReadAccess;
	BOOLEAN WriteAccess;
	BOOLEAN DeleteAccess;
	BOOLEAN SharedRead;
	BOOLEAN SharedWrite;
	BOOLEAN SharedDelete;
	UINT32 Flags;
	UINT64 CurrentByteOffset;
	UINT32 Waiters;
	UINT32 Busy;
	WCHAR FilePath[256];
	BOOLEAN DiskVolumePresent;
	UINT16 DeviceObject_Type;
	UINT16 DeviceObject_Size;
	UINT32 DeviceObject_RefCount;
	PVOID DriverObject;
	UINT32 DeviceObject_Flags;
	UINT32 DeviceObject_Characteristics;
	UINT32 DeviceObject_DeviceType;
	WCHAR DiskVolumePath[256];
}HUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE, * PHUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE;
```

This is straight-up copied from `FILE_OBJECT`, with exception to `FilePathPresent` and `DiskVolumePresent` If we get back to the usermode application, and after applying the previous struct as well, we will see that it checks for `FilePathPresent` and `DiskVolumePresent` before processing `FilePath` and `DiskVolumePresent`, here `FILE_OBJECT.FileName.Buffer` will contain the file path but without the physical disk volume its located on, so something like `\Users\ahm3dgg\tmp\test.exe`, and it also queries the the disk volume object manager path by calling into `ObQueryNameString` passing the device object associated with the file object, the returned path will look like `\Device\HarddiskVolume3`, these two paths are then combined in the usermode application and translated into something like `C:\Users\ahm3dgg\tmp\test.exe`

```cpp
  Result = DeviceIoControl(*(LockHunterCtx + 8), dwIoControlCode, &InBuffer, 8u, &OutBuffer, 0x446u, &BytesReturned, 0);
  vars48 = Result;
  if ( Result )
  {
    if ( OutBuffer.FilePathPresent || OutBuffer.DiskVolumePresent )
    {
      CopyString(DiskVolumePath, OutBuffer.DiskVolumePath);
      CopyString(FilePath, OutBuffer.FilePath);
      return 1;
    }
    else
    {
      return 0;
    }
  }
```

We can also see that it reads 0x446 = 1094 bytes, which is the size of our struct, the driver also checks for that, and also checks that the input buffer is of size 8 bytes.

So now we know the purpose of this driver, the driver is basically used to query file object related information, and the most important ones are the file path and the physical disk volume path in the object manager, the usermode application can then basically examine the locked file path against this information and it will also know that process owning that locked file through the handle entry `UniqueProcessId`

the check of the locked file path against the received file path, can be found at `sub_00646110`, this function is called directly after calling into the driver, its a little hard to see what its doing statically, since its relies on some context struct that I haven't really reversed, however I hooked up and the debugger, and started stepping through the function, in a hope of seeing the locked file path in showing up somewhere in the memory, and yes I found it, its taking two arguments the locked file path and the received file path from the driver, if you looked inside this function you will see that its calling `CompareStringW` passing to it `NORM_IGNORECASE` thus making a case-insensitive comparison.

![[Pasted image 20260226215634.png]]

if `sub_00646110` succeed which I named `CheckLockedFilePathAganistFilePath`, the function then loops through all processes on the system, using the information it got from `NtQuerySystemInformation` passing to it `SystemProcessInformation`, we have seen this above but we didn't discuss it.

```cpp
	ProcessLockingFile = ProcessInformation;
	while ( LODWORD(ProcessLockingFile->UniqueProcessId) != *&SystemInformation->Handles[i].UniqueProcessId )
	{
	  ProcessLockingFile = (ProcessLockingFile + ProcessLockingFile->NextEntryOffset);
	  if ( !ProcessLockingFile->NextEntryOffset )
		goto LABEL_30;
	}
	sub_40D820(&vars98, ProcessLockingFile->ImageName.Buffer);
```

at the end it calls into `sub_064185C`, passing to it the Process Lock File name, and the locked file path, I haven't digged into it, since its not really relevant I assume that it will build some sort of associative table to link Processes and the files they are locking.
### How LockHunter Closes Handles ?

Again, as with the question of how to query system handles, I actually didn't know how would you close a handle in a remote process, and I started reading through SystemInformer code, and also found this [Blog by Pavel Yosifovich](https://scorpiosoftware.net/2020/03/15/how-can-i-close-a-handle-in-another-process/)

The secret lies in `NtDuplicateObject`, which has the following definition

```cpp
NTSYSCALLAPI
NTSTATUS
NTAPI
NtDuplicateObject(
    _In_ HANDLE SourceProcessHandle,
    _In_ HANDLE SourceHandle,
    _In_opt_ HANDLE TargetProcessHandle,
    _Out_opt_ PHANDLE TargetHandle,
    _In_ ACCESS_MASK DesiredAccess,
    _In_ ULONG HandleAttributes,
    _In_ ULONG Options
    );
```

`NtDuplicateObject`, will duplicate a handle from one process into another process, the handle to be duplicated is at `SourceHandle`, `NtDuplicateObject` has an interesting flag called `DUPLICATE_CLOSE_SOURCE` which is according to ntdoc

> DUPLICATE_CLOSE_SOURCE: instructs the system to close the source handle. Note that this occurs regardless of any error status returned. The target handle parameter becomes optional when using this flag.

LockHunter calls this function, at `sub_0643460` which I renamed to `CloseRemoteHandle`

```cpp
__int64 __fastcall CloseRemoteHandleInternal(HANDLE SourceHandle, DWORD SourcePID)
{
  __int64 v2; // r8
  __int64 v3; // r9
  __int64 _0[5]; // [rsp+0h] [rbp+0h] BYREF
  __int64 *vars48; // [rsp+48h] [rbp+48h]
  unsigned __int8 vars5F; // [rsp+5Fh] [rbp+5Fh]
  HANDLE SourceProcessHandle; // [rsp+60h] [rbp+60h]
  NTSTATUS vars6C; // [rsp+6Ch] [rbp+6Ch]

  vars48 = _0;
  SourceProcessHandle = OpenProcess_0(0x1FFFFFu, 0, SourcePID);
  if ( NtDuplicateObject(SourceProcessHandle, SourceHandle, 0, 0, 0, 0, DUPLICATE_CLOSE_SOURCE) )
  {
    vars5F = 0;
    sub_643500(0, vars48, v2, v3, _0[4]);
  }
  else
  {
    vars6C = NtClose(SourceProcessHandle);
    return vars6C == 0;
  }
  return vars5F;
}
```

We can see that it first opens a handle to the source process, and then calls `NtDuplicateObject` passing `DUPLICATE_CLOSE_SOURCE`, thus closing the source handle, no need to specify the target handle when you just want to close the source handle.

I then went and Launched LockHunter on a locked file, and chose `Unlock Selected Process`, and then attached to LockHunter using x64dbg

after hitting the breakpoint, I examined the arguments 

![[Pasted image 20260226230540.png]]

We can use SystemInformer to help us with what we are seeing 

![[Pasted image 20260226230702.png]]

We can see that the handle value is `0xA8`, that's equal to the one we can see in rdx, after returning from `NtDuplicateObject`, the handle should be closed and LockHunter will present its Green Checkmark, but we for sure know that the file is not unlocked yet.

Now Enough Reverse Engineering, I am sure you may have already been able to attack the program without digging that deep, and in fact I did so, but I decided to make the blog longer and more informative :)
### Planning The Attack

I am pretty sure, you now know the problem we are trying to solve here, the issue with LockHunter is that it only looks for handles that refers to files, but files can be associated with sections as well, so how can I go about this ?

For the Usermode part, we can basically introduce some hook point or a patch whatever you wanna call it, that will not only check for File Handles, but Section Handles as well, that way the usermode application will send Both File/Section Objects to the driver to examine.

For the Kernel Mode part, I have rewritten the driver so that it can work with Section Objects and not just File Objects, but first let me explain some of the limitations my driver has, remember when we found out that when we actually start mapping the section using `NtMapViewOfSection`, even if we closed the file handle and the section handle, the file will still be locked. I haven't really implemented that because its not very trivial honestly closing a handle is not the same as unmapping a section view, and it will require many changes in the usermode application, that wasn't really in my scope, So I will stick here with just introducing visibility in open Section Handles beside File Handles, its still going to be useful, suppose a section view was unmapped but the handle wasn't closed, it will still catch that, after all I was doing that for the sake of learning, so I am not expecting anyone to use it.
### Hooking The Usermode Application

So Were can you introduce that hook ? if you took a look at the comparison of the object type index `ScanForLockingHandles` function

```c
.text:0000000000641601                 movzx   rax, byte ptr [rax+rcx*8+0Ch]
.text:0000000000641607                 cmp     al, [rbp+FileObjectTypeIndex]
.text:000000000064160D                 jnz     loc_641861
```

We can this its grabbing the `HandleEntry.ObjectTypeIndex` and then comparing it against `FileObjectTypeIndex`, if they are not equal it jumps to `loc_641861`.

So maybe we can just patch that jump, so that it jumps to our code, so that if the handle wasn't a file handle, we can trying comparing against section object type index, and if it was we can jump to the instruction just after `jnz loc_641861`, if not we will jump to `loc_641861` meaning its neither a file or a section handle.

The Stub should look something like this

```c
cmp al, 42       // Comparing aganist Section Object Type Index (Changes between windows different versions)
jz <location_after_previous_jz>  // Process the handle entry
jmp <next_handle_entry>          // handle is neither file or section, so skip it
```

So how are we going to inject our hook ? I decided to go with DLL Hijacking, I launched procmon and searched for status error `NAME_NOT_FOUND`, and found multiple entries one of them is `version.dll`, so I decided to go with it, we can just place it next to LockHunter and it will load it, but we also want to make sure we don't break the application, so we should proxy all the calls to the original `version.dll`, I will let you try to reason about the code, I suggest you also use `ZydisInfo.exe` to decode the instructions and understand there structure, that's what I did.

```cpp
#pragma comment(linker,"/export:GetFileVersionInfoA=C:\\Windows\\System32\\version.GetFileVersionInfoA,@1")
#pragma comment(linker,"/export:GetFileVersionInfoByHandle=C:\\Windows\\System32\\version.GetFileVersionInfoByHandle,@2")
#pragma comment(linker,"/export:GetFileVersionInfoExA=C:\\Windows\\System32\\version.GetFileVersionInfoExA,@3")
#pragma comment(linker,"/export:GetFileVersionInfoExW=C:\\Windows\\System32\\version.GetFileVersionInfoExW,@4")
#pragma comment(linker,"/export:GetFileVersionInfoSizeA=C:\\Windows\\System32\\version.GetFileVersionInfoSizeA,@5")
#pragma comment(linker,"/export:GetFileVersionInfoSizeExA=C:\\Windows\\System32\\version.GetFileVersionInfoSizeExA,@6")
#pragma comment(linker,"/export:GetFileVersionInfoSizeExW=C:\\Windows\\System32\\version.GetFileVersionInfoSizeExW,@7")
#pragma comment(linker,"/export:GetFileVersionInfoSizeW=C:\\Windows\\System32\\version.GetFileVersionInfoSizeW,@8")
#pragma comment(linker,"/export:GetFileVersionInfoW=C:\\Windows\\System32\\version.GetFileVersionInfoW,@9")
#pragma comment(linker,"/export:VerFindFileA=C:\\Windows\\System32\\version.VerFindFileA,@10")
#pragma comment(linker,"/export:VerFindFileW=C:\\Windows\\System32\\version.VerFindFileW,@11")
#pragma comment(linker,"/export:VerInstallFileA=C:\\Windows\\System32\\version.VerInstallFileA,@12")
#pragma comment(linker,"/export:VerInstallFileW=C:\\Windows\\System32\\version.VerInstallFileW,@13")
#pragma comment(linker,"/export:VerLanguageNameA=C:\\Windows\\System32\\version.VerLanguageNameA,@14")
#pragma comment(linker,"/export:VerLanguageNameW=C:\\Windows\\System32\\version.VerLanguageNameW,@15")
#pragma comment(linker,"/export:VerQueryValueA=C:\\Windows\\System32\\version.VerQueryValueA,@16")
#pragma comment(linker,"/export:VerQueryValueW=C:\\Windows\\System32\\version.VerQueryValueW,@17")

#include <phnt_windows.h>
#include <phnt.h>

#define IMAGE_LOCATE_NT_HEADERS(base) PIMAGE_NT_HEADERS(SIZE_T(base) + PIMAGE_DOS_HEADER(base)->e_lfanew)

#pragma pack(1)
struct Stub
{
	UINT8 _00[2] = { 0x3C, 0x2A };		// cmp al, 42
	UINT8 _01[2] = { 0x0F, 0x84 };		// jz
	UINT32 jz_relative_disp;			// (0x241613 - (base + 2))
	UINT8 _02[1] = { 0xE9 };			// jmp
	UINT32 jmp_relative_disp;			// (0x241861 - (base + 8))
};

void WriteExecutableCode(PVOID Base, PVOID Code, SIZE_T Size)
{
	DWORD OldProtection;
	VirtualProtect(Base, Size, PAGE_EXECUTE_READWRITE, &OldProtection);
	RtlCopyMemory(Base, Code, Size);
	VirtualProtect(Base, Size, OldProtection, &OldProtection);
}

void InstallHooks(PVOID ModuleBase)
{
	Stub StubV;

	auto BaseOfCode = PVOID(SIZE_T(ModuleBase) + IMAGE_LOCATE_NT_HEADERS(ModuleBase)->OptionalHeader.BaseOfCode);
	auto SizeOfCode = SIZE_T(SIZE_T(ModuleBase) + IMAGE_LOCATE_NT_HEADERS(ModuleBase)->OptionalHeader.SizeOfCode);
	PVOID Codecave = VirtualAlloc(nullptr, sizeof(StubV), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

	StubV.jz_relative_disp = UINT32((SIZE_T(ModuleBase) + 0x241613) - (SIZE_T(Codecave) + 2)) - 6;
	StubV.jmp_relative_disp = UINT32((SIZE_T(ModuleBase) + 0x241861) - (SIZE_T(Codecave) + 8)) - 5;

	RtlCopyMemory(Codecave, &StubV, sizeof(StubV));

	// 0F 85 4E 02 00 00    jnz     loc_241861
	auto PatchLocation = (UINT8*)(SIZE_T(ModuleBase) + 0x24160D);
	auto Disp = UINT32(SIZE_T(Codecave) - SIZE_T(PatchLocation)) - 6;
	WriteExecutableCode(PatchLocation + 2, &Disp, sizeof(Disp));
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD  ul_reason_for_call, LPVOID lpReserved)
{
	switch (ul_reason_for_call)
	{
	case DLL_PROCESS_ATTACH:
	{
		PVOID LockHunterBase = GetModuleHandle(NULL);
		InstallHooks(LockHunterBase);
	}
	case DLL_THREAD_ATTACH:
	case DLL_THREAD_DETACH:
	case DLL_PROCESS_DETACH:
		break;
	}
	return TRUE;
}
```
### Coding The Driver

[Kernel Driver Github Link](https://github.com/ahm3dgg/HunterxHunter/tree/main)

Now for the Fun part ! In the following section I will be describing how I went about reimplementing the kernel driver, and the different design decisions I have taken, so its my first time writing a kernel driver so I might have written bad code.

I won't be presenting the full code of the driver, however I will be presenting the relevant parts.

Here is the modified IRP Handle for Querying File Object Information

```cpp
NTSTATUS NTAPI HunterQueryFileInfoByPointer(PVOID DeviceContext, PIRP Irp, PIO_STACK_LOCATION IoStackLocation)
{
	UNREFERENCED_PARAMETER(DeviceContext);

	PHUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE SystemBuffer;
	PDEVICE_OBJECT DeviceObject;
	PFILE_OBJECT FileObject;
	PVOID Object;
	POBJECT_TYPE ObjectType;
	POBJECT_NAME_INFORMATION ObjectNameInfo;
	ULONG ReturnLength;
	UINT32 Length; 
	UINT32 ObjectNameLength; 
	NTSTATUS Status;

	SystemBuffer = Irp->AssociatedIrp.SystemBuffer;
	if ( IoStackLocation->Parameters.DeviceIoControl.InputBufferLength != sizeof(UINT64) || IoStackLocation->Parameters.Read.Length != sizeof(HUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE))
	{
		return STATUS_INVALID_PARAMETER;
	}

	Object = *(PVOID*)Irp->AssociatedIrp.SystemBuffer;
	ObjectType = ObGetObjectType(Object);
	
	if (ObjectType == *IoFileObjectType)
	{
		FileObject = (PFILE_OBJECT)Object;
	}
	else if(ObjectType == *MmSectionObjectType)
	{
		FileObject = MmGetFileObjectForSection(Object);
		
		if (!FileObject)
		{
			return STATUS_INVALID_PARAMETER;
		}
	}
	else
	{
		return STATUS_INVALID_PARAMETER;
	}

	SystemBuffer->FileNamePresent = FALSE;
	SystemBuffer->DiskVolumePresent = FALSE;

	if (FileObject->Type != 5)
	{
		return STATUS_INVALID_PARAMETER;
	}

	SystemBuffer->Type = FileObject->Type;
	SystemBuffer->Size = FileObject->Size;
	SystemBuffer->DeviceObject = FileObject->DeviceObject;
	SystemBuffer->LockOperation = FileObject->LockOperation;
	SystemBuffer->DeletePending = FileObject->DeletePending;
	SystemBuffer->ReadAccess = FileObject->ReadAccess;
	SystemBuffer->WriteAccess = FileObject->WriteAccess;
	SystemBuffer->DeleteAccess = FileObject->DeleteAccess;
	SystemBuffer->SharedRead = FileObject->SharedRead;
	SystemBuffer->SharedWrite = FileObject->SharedWrite;
	SystemBuffer->SharedDelete = FileObject->SharedDelete;
	SystemBuffer->Flags = FileObject->Flags;
	SystemBuffer->CurrentByteOffset = FileObject->CurrentByteOffset.QuadPart;
	SystemBuffer->Waiters = FileObject->Waiters;
	SystemBuffer->Busy = FileObject->Busy;

	if (FileObject->FileName.Length <= sizeof(SystemBuffer->FilePath))
	{
		Length = FileObject->FileName.Length;
	}
	else
	{
		Length = sizeof(FileObject->FileName);
	}

	RtlZeroMemory(SystemBuffer->FilePath, sizeof(SystemBuffer->FilePath));
	RtlCopyMemory(SystemBuffer->FilePath, FileObject->FileName.Buffer, Length);
	SystemBuffer->FileNamePresent = TRUE;
	DeviceObject = FileObject->DeviceObject;
	
	if (DeviceObject->Type == 3)
	{
		SystemBuffer->DeviceObject_Type = DeviceObject->Type;
		SystemBuffer->DeviceObject_Size = DeviceObject->Size;
		SystemBuffer->DeviceObject_RefCount = DeviceObject->ReferenceCount;
		SystemBuffer->DriverObject = DeviceObject->DriverObject;
		SystemBuffer->DeviceObject_Flags = DeviceObject->Flags;
		SystemBuffer->DeviceObject_Characteristics = DeviceObject->Characteristics;
		SystemBuffer->DeviceObject_DeviceType = DeviceObject->DeviceType;
		RtlZeroMemory(SystemBuffer->DiskVolumePath, sizeof(SystemBuffer->DiskVolumePath));

		ObjectNameInfo = ExAllocatePool2(POOL_FLAG_PAGED, 0x1000, HUNTER_POOL_TAG);
		if (!ObjectNameInfo)
		{
			return STATUS_INSUFFICIENT_RESOURCES;
		}

		Status = ObQueryNameString(DeviceObject, ObjectNameInfo, 0x1000, &ReturnLength);
		if (!NT_SUCCESS(Status))
		{
			return Status;
		}
		
		if (ObjectNameInfo->Name.Length <= sizeof(SystemBuffer->DiskVolumePath))
		{
			ObjectNameLength = ObjectNameInfo->Name.Length;
		}
		else
		{
			ObjectNameLength = sizeof(SystemBuffer->DiskVolumePath);
		}

		RtlCopyMemory(SystemBuffer->DiskVolumePath, ObjectNameInfo->Name.Buffer, ObjectNameLength);
		ExFreePool(ObjectNameInfo);
		SystemBuffer->DiskVolumePresent = TRUE;
	}

	Irp->IoStatus.Information = sizeof(HUNTER_QUERY_FILE_OBJECT_INFO_RESPONSE);
	return STATUS_SUCCESS;
}
```

Its the same as the one we saw in the original driver, but with some minor changes.

First we check to the object type using an undocumented function `ObGetObjectType` this function is exported however its not documented in the WDK, every executive object in windows has a type, which is actually a object in of it self, called `OBJECT_TYPE`, `OBJECT_TYPE` holds static information that are same for all instances of a specific object, it also links objects of same type together, this was used to save memory so that you don't have to embed a static information, in every object's `OBJECT_HEADER`.

If the object type is a file object, we just set `FileObject` to be the `Object` we received, however when its a `Section`, here I had to go for two routes one of them uses the documented exported functions and the other uses undocumented unexported function, but is so much easier.

The First approach works as follows

- Get a Handle To Section Object using `ObOpenObjectByPointer`
- Map The Section using `NtMapViewOfSection`
- Use `NtQueryVirtualMemory` `MemoryMappedFileName` Info Class, to get Memory Mapped FileName (I learned this from SystemInformer)
- Use `ZwCreateFile` to open a handle to the file given the filename received from `NtQueryVirtualMemory`
- Use `ObReferenceObjectByHandle` to get a file object pointer

Sounds tedious right ? I mean it makes no sense to acquire a handle to an object I already have a direct pointer at, and also the `PFILE_OBJECT` is infact embedded in the `SECTION_OBJECT`, but its an undocumented structure that changes between different windows versions, and I didn't really want to do it.

So I opened `ntoskrnl.exe`, and soon I found `MmGetFileObjectForSection`, sounds exactly what I want, why isn't that exported ? its very useful !

This function is called from one single place `FsRtlCreateSectionForDataScan`, which is documented and exported but it doesn't do what we want.

So I decided to pattern match on this function, and use it, as you can see in the code it made our life much easier, all what we have to do is to call this function, give it the section pointer and it will give us back the file object pointer, yes I know this I know this way is also undocumented and may break in the future, but Idk I have a feeling that its less likely to break, than maintaining the different versions of `SECTION` structs.

Also, There is one case where it will return null, this is when the section is page-file backed, so it doesn't refer to an actual file on disk, I make sure to check for that.

Here The code I use to grab `MmGetFileObjectForSection`, you can find the patterns as well as the full source code in the [GitHub Repo](https://github.com/ahm3dgg/HunterxHunter) associated with this blog 

```cpp
PVOID GetKernelBase(PDRIVER_OBJECT DriverObject)
{
	PVOID KernelBase = NULL;
	PLIST_ENTRY ModuleEntryListHead;
	PLIST_ENTRY ModuleListLink;

	ModuleEntryListHead = (PLIST_ENTRY)DriverObject->DriverSection;
	ModuleListLink = ModuleEntryListHead->Flink;

	while (ModuleListLink != ModuleEntryListHead)
	{
		PLDR_DATA_TABLE_ENTRY ModuleEntry = CONTAINING_RECORD(ModuleListLink, LDR_DATA_TABLE_ENTRY, InLoadOrderLinks);

		if (RtlEqualUnicodeString(&ModuleEntry->BaseDllName, &(UNICODE_STRING)RTL_CONSTANT_STRING(L"ntoskrnl.exe"), TRUE))
		{
			KernelBase = ModuleEntry->DllBase;
			break;
		}

		ModuleListLink = ModuleListLink->Flink;
	}

	return KernelBase;
}

NTSTATUS GetGoldFromMemory(PDRIVER_OBJECT DriverObject)
{
	gKernelBase = GetKernelBase(DriverObject);
	PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)gKernelBase;
	PIMAGE_NT_HEADERS nt = (PIMAGE_NT_HEADERS)((SIZE_T)gKernelBase + dos->e_lfanew);
	
	SIZE_T MmGetFileObjectForSection_CallPatternAddress = (SIZE_T)FindPattern(
		(UINT8*)&FsRtlCreateSectionForDataScan,
		nt->OptionalHeader.SizeOfCode,
		MmGetFileObjectForSection_CallPattern, 
		sizeof(MmGetFileObjectForSection_CallPattern), 
		MmGetFileObjectForSection_CallPatternMask
	);

	if (!MmGetFileObjectForSection_CallPatternAddress)
	{
		DbgPrint("[ERROR]: Failed To Find MmGetFileObjectForSection\n");
		return STATUS_UNSUCCESSFUL;
	}

	SIZE_T MmGetFileObjectForSection_CallInstAddr = MmGetFileObjectForSection_CallPatternAddress + 26;
	UINT32 MmGetFileObjectForSection_RVA = *(UINT32*)(MmGetFileObjectForSection_CallPatternAddress + 27);
	UINT8* pMmGetFileObjectForSection = (UINT8*)(MmGetFileObjectForSection_CallInstAddr + MmGetFileObjectForSection_RVA + 5);

	if (!FindPattern(
		pMmGetFileObjectForSection, 
		nt->OptionalHeader.SizeOfCode, 
		MmGetFileObjectForSectionPattern, 
		sizeof(MmGetFileObjectForSectionPattern), 
		MmGetFileObjectForSectionPattern_Mask
	))
	{
		DbgPrint("[ERROR]: Failed To Find MmGetFileObjectForSection\n");
		return STATUS_UNSUCCESSFUL;
	}

	MmGetFileObjectForSection = (FPT_MmGetFileObjectForSection)pMmGetFileObjectForSection;
	DbgPrint("[INFO]: Found MmGetFileObjectForSection at %p\n", MmGetFileObjectForSection);

	return STATUS_SUCCESS;
}
```

- First we get the kernel base address

- We then call `FindPattern`, and start searching from the beginning of `FsRtlCreateSectionForDataScan`, I call this pattern `MmGetFileObjectForSection_CallPattern`

- I then double check to see if the function we got is actually `MmGetFileObjectForSectionPattern`, by matching against another pattern, I call it `MmGetFileObjectForSectionPattern`

So Now After replacing the original driver with our driver, 

Before replacing the driver we had 

![[Pasted image 20260227092701.png]]

And After

![[Pasted image 20260227092848.png]]

Now when choosing `Unlocked Selected Process`, it will close both the file and section handle.

### Debugging The Project

While Coding and Debugging this project, I faced numerous amount of BSODs, also since the usermode application was sending an IOCTL for every single opened file about 16000 on my system, I couldn't compare properly what my driver was sending versus what the real driver was sending, for that I hooked `DeviceIoControl`, and wrote the output buffer in a file, then I let Claude write for me a 010 editor template for parsing the structure, because honestly I don't have time for learning 010 scripting, I was already having a lot of things to code my self :), Using this template helped me identify an issue with the data I was sending.

Anyways, here is it and also I think 010 editor is a great Tool !

```c
//------------------------------------------------
//--- 010 Editor v1.0 Binary Template
// File: HUNTER_FIND_FILENAME_RESPONSE.bt
// Purpose: Parse array of HUNTER_FIND_FILENAME_RESPONSE structs
//------------------------------------------------

// Type definitions
typedef byte    BOOLEAN;
typedef uint16  UINT16;
typedef uint32  UINT32;
typedef uint64  UINT64;
typedef uint64  PVOID;  // 64-bit pointer (change to uint32 for 32-bit targets)
typedef wchar_t WCHAR;

typedef struct {
    BOOLEAN     FileNamePresent;
    UINT16      Type;
    UINT16      Size;
    PVOID       DeviceObject;
    BOOLEAN     LockOperation;
    BOOLEAN     DeletePending;
    BOOLEAN     ReadAccess;
    BOOLEAN     WriteAccess;
    BOOLEAN     DeleteAccess;
    BOOLEAN     SharedRead;
    BOOLEAN     SharedWrite;
    BOOLEAN     SharedDelete;
    UINT32      Flags;
    UINT64      CurrentByteOffset;
    UINT32      Waiters;
    UINT32      Busy;
    WCHAR       FileName[256];
    BOOLEAN     ObjectNamePresent;
    UINT16      DeviceObject_Type;
    UINT16      DeviceObject_Size;
    UINT32      DeviceObject_RefCount;
    PVOID       DriverObject;
    UINT32      DeviceObject_Flags;
    UINT32      DeviceObject_Characteristics;
    UINT32      DeviceObject_DeviceType;
    WCHAR       ObjectName[256];
} HUNTER_FIND_FILENAME_RESPONSE <read=ReadEntry>;

// Display function - shows filename in the tree view
string ReadEntry(HUNTER_FIND_FILENAME_RESPONSE &e) {
    if (e.FileNamePresent)
        return WStringToString(e.FileName);
    return "<no filename>";
}

// Calculate number of entries from file size
local uint64 entrySize = sizeof(HUNTER_FIND_FILENAME_RESPONSE);
local uint64 numEntries = FileSize() / entrySize;

// Parse the array
HUNTER_FIND_FILENAME_RESPONSE entries[numEntries] <optimize=true>;
```
### Potential DOS Vulnerability in the Driver

To end this blog, since its really long now, there is actually a Denial of Service vulnerability in the driver, that can be used to trigger a BSOD (Blue Screen of Death), if you went back to the original driver, you will see that it trusts the usermode application, and assumes that it will only receive File object pointers, and then starts reading data from it, this is wrong, the author should have found a way to check for the received file object, maybe like we did using `ObGetObjectType`, and make sure its a File Object before proceeding, and if not return a `STATUS_INVALID_PARAMETER`, not doing though will cause a `PAGE_FAULT_IN_NONPAGED_AREA` exception.

![[Pasted image 20260227074538.png]]

That's it.

~ ahm3dgg