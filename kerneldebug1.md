# Kernel Debugging of a Windows 2012 R2 (or 8.1) guest on VirtualBox, from a Windows 10 Host
## TL;DR
This is a post detailing how to set up VirtualBox (version 5.2)  running on Windows 10, to do kernel debugging of Windows 2012 (or 8.1), using WinDbg.
It is very specific because all things kernel related are, I can not make *any* promises about any of this working with other versions of any of the components mentioned wortwith.

## Required Pieces
- VirtualBox 
- A Windows Server 2012 R2 or 8.1 guest OS
- WinDbg

## Setup 
I will assume VirtualBox is installed, and I installed it using defaults and no special options. 
Create a VM for Windows 2012 R2 (or 8.1) with the memory and disk space that suits you, my choices were

- 20 Gig drive
- 2 Gig RAM
- 2 Cores

!!! note
    Start up and install it, then do the following, or do the following, then install, it doesn't really matter.

Configure the VM's network to be *Bridged*. 
Set up a Serial port as follows;

- Port number: COM1
- Port Mode: Host Pipe
- Path/Address: ```\\.\pipe\<WHATEVERYOULIKE>```
- Do *not* tick ```Connect to existing pipe/socket```

When you now start the VM the pipe will be created and this is where you will ultimately connect WinDbg. 

But, first, in your VM guest session; launch a ```cmd``` shell as administrator and set up the kernel debugging options;

- ```bcdedit /debug on```
- ```serial debugport:1 baudrate:115200```

At this point you should restart the VM. This will create the named pipe, connected to COM1 in the guest Windows session.

Now all that is left is to connect with WinDbg from the host;

```Windbg -b -k com:pipe,port=\.\pipe\windebugpipe,resets=0,reconnect```

This will bring up a window saying "waiting to connect", and you now have to restart the VM session. Once it boots the debugger will connect (assuming everything is working) and you'll be able to break into the Windows kernel and start snooping around.

## References
- [Advanced Windows Debugging](https://www.amazon.co.uk/Advanced-Windows-Debugging-Administering-Addison-Wesley/dp/0321374460)
- [Windows Internals](https://www.amazon.co.uk/Windows-Internals-Part-architecture-management/dp/0735684189)