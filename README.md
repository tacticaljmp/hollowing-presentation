This readme provides some technical information about process hollowing in general, as well as how hollowing is implemented in some tools I'm involved in. Further below there is an experimental section where some tests against AV solutions are presented.
The pdf contains some legacy slides about process hollowing. 


**TLDR:**
Process hollowing spawns a target process and replaces it's code with your payload:

	- Create suspended process  
	- Get image base from target PEB  
	- Unmap old target image
	- Allocate new memory in target process
	- Apply relocations to payload image
	- Copy payload image into target memory
	- Set allocated memory start address as new target image base
	- Overwrite target entry point to execute payload
	- Resume target main thread
	
	PRO:
	* Can disguise payload as legitimate process (vs. IDS/IPS)
	
	CON:
	* "Old" technique, likely recognized by real-time protection


The [BFG project](https://github.com/govolution/bfg) covers various injection techniques - process hollowing is one of them. The general idea behind process
hollowing is to replace all (or part) of the executable data of a target process with our own payload.
Main advantage of this is a disguise bonus: Our payload now runs in the context of the hollowed process. 
For example, we could create a new instance of C:\windows\system32\svchost.exe and hollow our payload into it. At the time of process
creation, it looks like a legitimate svchost instance is launched. However, we suspend the process and hollow our payload into it.
From the OS's point of view, it's still svchost running, but instead of some neat windows service magic our payload is now executed.
Now that we're svchost (at least at first glance), the OS or not-so-nitpicky security solutions probably won't mind if we send/receive some network packets
or do other fancy stuff that would be worthy of an svchost. 
An obvious drawback of proceeding like this is that the original functionality of the target process is lost, since we abandoned the old executable data.
This is in contrast to other techniques such as classic dll injection, where we launch our payload as a new thread in the target process. Doing so, the
original functionality of the target remains up and running.
In the following, we'll explain some technical details about how process hollowing is implemented in this tool.

As mentioned earlier, the first step is to create a new instance of our desired target process by calling CreateProcess. Important to note here
is that the process is created in a suspended state. This gives us the opportunity to inject our payload.
In the current state, the target process has already been mapped into memory by the windows loader. With ASLR present, the image base of the target is
dynamic and randomized by the OS. In order to find out at which base address the target image is mapped, we call ReadProcessMemory to get that
information from the target's Process Environment Block (PEB). We acquire the address of the PEB from the target process CONTEXT, which contains the saved register values
and is accessible via GetThreadContext.
Next, we (optionally) unmap the original target image by calling NtUnmapViewOfSection. Optionally means that since we control how our new payload image will be mapped
into the target process address space, we can map around the old image and just leave it be. Some tests against AV solutions hinted that not calling NtUnmapViewOfSection
may help with appearing a little more stealthy, as the missing function call makes it look less like process hollowing in behavioural analysis.
Now we prepare our payload image, which is given as raw PE (exe) data input by the tool. First, all sections of the image are copied into a local buffer. After that, we allocate a new area
of memory in the target process. The base address of this new memory area will later be the new image base of the target process, and hold our payload.
As this new base address will very unlikely equal the desired image base of our payload PE, we need to calculate the address difference and apply relocations.
The payload is now prepared in our local buffer, so the next step is to copy our local buffer into the target process address space newly allocated by us.
To make the target process execute our payload, we adjust the control flow by resetting the entry point in the respective CONTEXT register and overwrite the old image base with our new
image base in the target PEB.
Finally, we can start execution by resuming the target main thread.
If you are interested in further implementation details, please refer to the source code.

Process hollowing, as described here, is a somewhat "old" technique and, as such, not quite top-notch regarding AV-stealthiness. In fact, the used sequence of API calls is well-known and will
likely be caught by real-time protection mechanisms. In the supplied BFG build scripts, payload obfuscation is configured by default. However, hollowing as a deployment technique may need further
disguise for it's own, depending on the defense sophistication level of your target.
But remember: BFG is "not meant to be another antivirus evasion tool".

**FYI** a more "innovative" hollowing technique: "Process Doppelgänging" (featured at BlackHat EU '17)
https://www.blackhat.com/docs/eu-17/materials/eu-17-Liberman-Lost-In-Transaction-Process-Doppelganging.pdf
(Will probably get implemented in BFG some day...)

### Evaluation against AV

To somehow quantify how hollowing in BFG fares against AV, I conducted some small tests in virtual machines. For the VMs, I used the Windows IE developer images, with Windows 7 for x86 and Windows 10 for x64, respectively.
For each architecture, I set up four VMs with trial AV endpoint solutions:
- Windows Defender only
- McAfee
- Kaspersky Free
- Sophos Home

The first test was based on the question to what extent BFG'ed payload were actually detected. For the tests I used the
`build_winXX_hollowing_revtcp_exe.sh` (where XX refers to the target architecture) build scripts. The payload used in these scripts is a reverse tcp meterpreter. The payload gets injected into a legitimate calc.exe copied to desktop. To draw a comparison, I tried executing the raw payload as well as the BFG'ed payload on each target.

| x64 (Win 10)     | Raw payload detected? | BFG'ed payload detected?
| ---------------- | --------------------- | ------------------------
| Windows Defender | Yes                   | Yes
| Mc Afee          | Yes                   | No
| Kaspersky Free   | Yes                   | No
| Sophos Home      | Yes                   | Yes

| x86 (Win 7)      | Raw payload detected? | BFG'ed payload detected?
| ---------------- | --------------------- | ------------------------
| Windows Defender | No                    | No
| Mc Afee          | Yes                   | No
| Kaspersky Free   | Yes                   | Yes
| Sophos Home      | Yes                   | Yes

As this result was not quite satisfying, my next question was: Will an innocent payload also be flagged? This would at least generate some insight if the payload raised suspicion, or rather the hollowing procedure itself.
So I did a second run, but this time I used the `build_winXX_hollowing_hello_exe.sh` scripts, assuming that popping a hello world message box would be adequately innocent.

| x64 (Win 10)     | Raw payload detected? | BFG'ed payload detected?
| ---------------- | --------------------- | ------------------------
| Windows Defender | No                    | No
| Mc Afee          | No                    | No
| Kaspersky Free   | No                    | No
| Sophos Home      | No                    | No

| x86 (Win 7)      | Raw payload detected? | BFG'ed payload detected?
| ---------------- | --------------------- | ------------------------
| Windows Defender | No                    | No
| Mc Afee          | No                    | No
| Kaspersky Free   | No                    | No
| Sophos Home      | No                    | Yes

This hinted that the applied BFG stuff was not the problem. The only hit being Sophos, which caught my sequence of API calls via real-time protection (well, at least that is what it advertised on the screen...). Interestingly enough, based on the state of how far my debug outputs progressed, this interception allowed for some, let's call it grey-box-testing. Since we know the precise location where Sophos intercepted, we also know which API call triggered the high-enough suspicion level. I did not thoroughly dig into this further, but further not-so-comprehensively-written-down tests hinted that not unmapping the old target process image via NtUnmapViewOfSection at least lowered the suspiciousness score.

However, the tests also indicated that the obfuscation rate was not sufficient. Though you can fill books with reasons why AVs flag binaries, the simplicity of the encoder used to disguise the payload (simple XOR) should likely be the main issue here.
It's worth noting that I did these tests round about in Mar/Apr '18 (if I recall correctly), so the first thing I did afterwards was to write an experimental encoder not solely based on XOR. So I included some variations, you can check it out in the BFG source code. In any case, it would be wise to integrate more AV evasion capabilities, e.g. by pre-chaining [AVET](https://github.com/govolution/avet) or other AV evasion tools.

Interestingly, some further "inofficial" tests indicated that detection rate fluctuates with changing AV updates. All of a sudden Kaspersky no longer flags the binary before execution, but detects the meterpreter connection as a network attack. It also seems that not all injection targets are equally monitored: Injection into svchost was flagged, injection into the deployed BFG executable however was not. I also had some cases where the meterpreter sessions worked cleanly despite presence of Sophos.
If these results originate from changes in BFG or on behalf of AV patches is not quite clear. Conducting further tests with a newer BFG version is on the TBD list.
