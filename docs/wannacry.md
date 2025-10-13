## WannaCry

In the early summer of 2017, WannaCry was unleashed on the world. Widely considered one of the most devastating malware infections to date, WannaCry left a trail of destruction in its wake. WannaCry is a classic ransomware sample; more specifically, it is a ransomware cryptoworm, meaning it can encrypt individual hosts and has the capability to propagate through a network on its own.

## Objective

Perform a full analysis of WannaCry and answer the questions below.

## Challenge Questions

- Record any observed symptoms of infection from initial detonation. What are the main symptoms of a WannaCry infection?
- Use FLOSS and extract the strings from the main WannaCry binary. Are there any strings of interest?
- Inspect the import address table for the main WannaCry binary. Are there any notable API imports?
- What conditions are necessary to get this sample to detonate?
- **Network Indicators**: Identify the network indicators of this malware
- **Host-based Indicators**: Identify the host-based indicators of this malware. 
- Use Cutter to locate the killswitch mechanism in the decompiled code and explain how it functions.

## Static Analysis

Static analysis examines the malware sample without executing it. This provides an early view into the sample’s structure, imports, strings, and other static indicators that suggest its capabilities and intent.


### Strings analysis

Extracting strings is a fast way to reveal hardcoded URLs, filenames, user-agent strings, or other indicators. Use `FLOSS` to extract and deobfuscate strings that are likely relevant:

```bash
cd /path/to/sample
floss.exe -n 8 WannaCry.exe > strings.txt
```

Review `strings.txt` for potential kill-switch domains, download URLs, file paths, or recognizable crypto-related identifiers.

### Floss
![floss](assets/img/floss0.png)

There are too many strings in the binary to check one by one. Some look random, but others contain useful information that can help us understand what the malware does.


![floss](assets/img/floss1.png)

We can notice a bunch of API calls, and it seems they are imported from other executables. One giveaway is the DOS header, which we will see a couple more times.

![floss](assets/img/floss3.png)

You can see Windows binaries like `icacls`, and the command `attrib +h .` suggests that it creates a hidden directory somewhere!

![floss](assets/img/floss2.png)


### PE Studio
Open the sample in `PE Studio` and inspect the Import Address Table. Check the **Indicators** view for notable items — in this case, PE Studio shows three packed executables embedded in the first-stage binary and it flags a URL.

Now check the **Libraries / Imports** section to see which DLLs and APIs are referenced. In this sample, the Windows Sockets API (e.g., `ws2_32.dll`) and Windows Internet extensions (WinINet, e.g., `wininet.dll`) appear in the imports. That combination indicates the binary likely performs socket/network operations and uses higher-level WinINet functions for HTTP requests or downloads. Follow-ups: note the exact imported functions (e.g., `InternetOpen`, `InternetConnect`, `URLDownloadToFile`) and mark them as primary candidates for breakpoints during dynamic analysis.

A helpful next step is to sort the Imports by risk or use a blacklist filter to surface high-risk APIs first. Near the top of the list, we see several CryptoAPI entries (for example, `CryptGenKey`, `CryptImportKey`, `CryptEncrypt`, `CryptDecrypt`) — consistent with ransomware behavior. This strongly suggests encryption operations are implemented in the binary. Recommended follow-ups: record the exact crypto function names, trace their call sites in the decompiler, and set breakpoints on the crypto APIs to capture keys and observe encryption behavior during execution.

![pe](assets/img/pe1.png)

![pe](assets/img/pe2.png)

Checking the section sizes: Raw = 3,719,168 bytes, Virtual = 6,718,034 bytes. The virtual size is larger by 2,998,866 bytes (~2.86 MiB), an increase of about 80.6% over the raw size. Such a large Raw→Virtual gap is a strong indicator of packing or embedded payloads (high entropy sections, unpacking at runtime, or multiple embedded PE blobs).

![pe](assets/img/pe3.png)

WannaCry will try to contact a specific URL (the long, weird one seen in strings and PE Studio). If it connects to this URL, the payload does not run.

Change the sample extension to .exe to arm it. Start INetSim and open Wireshark in Remnux.

![de](assets/img/det.png)

We see that after our TCP handshake, there is an HTTP packet that makes a request to `http://www.1uqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com/`

![wire](assets/img/wire.png)


Now inetsim responds with a 200 OK for this

![wire](assets/img/wire2.png)

But if we go back to Flare VM, we see that this `WannaCry` does not execute. It does not run if it gets a 200 OK from the callback URL.

So, to proceed, go back to `Remnux` and press `Ctrl+C` to stop `inetsim`.


In Flare VM, open `cmder` and flush the DNS resolver cache:

```bash
ipconfig /flushdns
```


**Therefore, the condition necessary to get the sample to detonate is that INetSim must be turned off.**



Now, to gather more network indicators for this sample, we can use tools on the endpoint itself.


Go to `SysinternalsSuite` (C drive) and find `TCPView`.


Run `TCPView` as Administrator. When you detonate the binary, you will see a lot of traffic going out to port 445 to remote addresses (for example, 169.254.130.1).


This means there is no real address connectivity—169.254.x.x is an auto-assigned IP address.


But notice all the network connections going out on port 445, which would normally be to different hosts on the network.


This shows how WannaCry tries to propagate itself. It is ransomware, but also a worm.


It spreads using the `EternalBlue` exploit, which targets Windows SMB. SMB uses port 445.


If you let the binary run for a while, you may see a new process called `taskhsvc.exe` appear, which opens a listening port on 1950.

![wire](assets/img/tcpv1.png)

![wire](assets/img/tcpv2.png)

## Host based indicators:


We’ll go over to `Procmon` (Process Monitor) and open the `Filter` tab.
We will filter on a few different things. We can filter for `Process Name contains wannacry`, which is our process, and we can start with `Operation is createfile`. Go ahead and hit OK and then run WannaCry as Administrator.

![proc](assets/img/procm1.png)


One of the first things that we notice is the creation of a file called `taskhsvc.exe` in `C:\Windows`. So we’ll go check that out.

![task1](assets/img/tasksche1.png)



Given that another executable is created from the initial executable, let’s go ahead and drill down on that.
One way to do that is by using the `Process Tree` (tab at the top).
It looks like from the original `WannaCry` binary, `taskhsvc.exe` is unpacked and then run with an argument of `/i`.



Now let’s take the PID (process ID) of the original ransomware binary, and we’ll add that as the parent PID to see if we can identify what `taskhsvc.exe` might be doing. So we will go to the filter tab, add ‘Parent PID is xxx’, hit add, and remove the other criteria given there.

![task2](assets/img/tasksche2.png)


Now we can see that there is a process started, and that’s the beginning of the execution of this secondary payload.


Let’s go ahead and filter for `Operation is createfile`, add it, and hit OK.

![task3](assets/img/tasksche3.png)


We see that in `C:\ProgramData`, there appears to be a strangely named directory.


So we can open this up and see that this directory is right here.

![task4](assets/img/tasksche4.png)



Opening this directory reveals a staging area for WannaCry’s execution and unpacking of all its packed resources. This is installed as a hidden directory using the `attrib +h` command we saw in static analysis.


**So this is another host-based indicator (the second stage of WannaCry installing as a hidden directory in `C:\ProgramData`).**



Now, when we look at service creation in Task Manager, in the Services tab, we can see a service created with the same strange name as the folder we just saw. This is the persistence mechanism.

![task4](assets/img/tasksche5.png)



This is the service that ensures if you restart the computer, WannaCry will start again and re-encrypt anything added to the host, such as a USB drive or new files.


Let’s take a look at the kill switch function:


Open the tool `Cutter`, start by finding the main function, and go to the graph mode view.
`Cutter` is a free and open-source GUI reverse engineering platform. It provides an interactive disassembler, decompiler, and debugger for analyzing binaries.


One of the first things we can see is that a string reference to this weird URL is loaded into `esi` right at the beginning of the program.
Therefore, a whole bunch of arguments are marshaled to make an API call.




![cutter](assets/img/cutter1.png)


The first API call made is `InternetOpenA`, which prepares to open a handle to a given web resource. At this point, the contents of `eax` are moved into `esi` and pushed onto the stack as well.


!!!note
    `ESI` is the source index register in x86 architecture. It is commonly used for string and memory operations, often serving as a pointer to source data in memory, but can also be used as a general-purpose register to hold values or pointers during function calls and data movement.

!!!note
    `EAX` is the primary accumulator register in x86 architecture. It is commonly used to store the result of operations or function return values, and is often used for passing data between instructions or API calls.

    
Then once we take a look at the `Decompiler tab` (down left), here the outcome of the `InternetopenA URL` is loaded into the register `eax` then it is loaded into the `edi` register.

!!!note
    `EDI` is the destination index register in x86 architecture. It is often used for operations involving memory copying, string manipulation, or as a general-purpose register to hold pointers or handles, especially in API call sequences.

![cutter](assets/img/cutter2.png)


Basically, `InternetOpenA` will return a binary value (1 or 0) indicating success or failure, which is then stored in the `edi` register.


Now, come back to the graph mode. There is a test instruction for `edi` to test itself, and then there will be a jump depending on the value of `edi`.

![cutter](assets/img/cutter3.png)


If `edi` is tested against itself and the value is zero, the Zero flag is set. Then, the `jne` (jump if not equal) instruction evaluates the Zero flag, and depending on its state, one of two things will happen.

So if the outcome of this API call is true—meaning it reaches out to the specified URL and the API call succeeds—we’re going to this location in memory.

![cutter](assets/img/cutter4.png)


On the other hand, if this API call reaches out to the URL and there is nothing there, the zero flag will indicate that we're going to jump to this other location.

![cutter](assets/img/cutter5.png)


This location is almost exactly the same, but there’s one difference: this function call right here.
If we trace into this, it is the rest of the encryption payload—this installs itself as a service.

![cutter](assets/img/cutter6.png)


This opens up and unpacks the rest of the resources in WannaCry’s binary and will kick off the encryption routine, severely impacting the computer it is run on.


So this is the kill switch URL function, and the routine goes like this:

- **Check the URL**
- **If there is a result, exit the program completely**
- **If there is no result, run the function call that executes the encryption and payload routine**


So now let’s try to execute the program even if it gets a result for that URL it calls out.
Now run INetSim on Remnux.
Then in Flare VM, go to `cmder` and flush the DNS resolver cache with:

```bash
ipconfig /flushdns
```

At this point, if we arm and detonate this binary, there should be no actual payload detonation because INetSim is up and running.
But let’s load this into a debugger!


Open the `x32dbg` debugger and attach it to the executable.
Now we want to find the main function, so hit `F9`.

![deb](assets/img/debug1.png)


Open up the `search` and search all modules for a string reference. Search for that weird URL (`http://www.1uqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com/`) in the search bar.
Then go ahead and set a breakpoint (toggle breakpoint).


The idea is that this is going to be loaded into that API call and pushed onto the stack at some point. So we want to find when that happens.
Then go to CPU view and hit F9 one time. It seems that this is the point at which our random weird URL string is moved into `esi`.

![deb](assets/img/debug2.png)


We can confirm this because both `InternetOpenA` and `InternetOpenUrlA` are the API calls right here.