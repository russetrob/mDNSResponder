## Compile:
```
Notable issues:

Solution stucture bit confusing:
    mDNSResponder.sln  seems to be main one.
        * mDNSNetMonitor
            * Montiors output MDNS I/O.
            * Depends on the mdnCore APIs without any deps.
            * Depends on mDNSResponder
            
Needed to add LOG_ERR option.

* Need MFC to build the XXX project.

* Need to remove "_LEGACY_NAT_TRAVERSAL_" macro from mDNSResponder.vcxproj
    * Don't know why it's there.
    * File from apple isn't present.
    * Don't need NAT traversal.
    
BonjourExample.sln

    command line  warning D9035: option 'Gm' has been deprecated and will be removed in a future release
        IGNORE FOR NOW.
    dnssd.lib has wrong realitve path.
        corredced.
    
    
BonjourQuickLooks.sln
    INCOMPATIBLE.
    
mDNSWindows\DNSServiceBrowser\Windows\ApplicationVS2003.sln
    mDNSWindows\DNSServiceBrowser\Windows\ApplicationVS2003.vcxproj
        Looks old.
        Tried upgrading, but 
    
    
Successful build of all projects in
    mDNSResponder.sln
    Bonjourexample.sln
```   
## The projects:

* mDNSResponder solution
    * DLLStub
        * Library to link with to access DLL.
    * dnssd
        * dnsd.dll
        * This is the client-side implementation of the bojour API.
        * Connects to mDnsResponder via localhost port 5354
    * dns-sd
        * Command line tool
        * Depends on dnssd.dll for access to API.
    * DLLX
        * Appears to be a COM interface to the dnssd library
    * mDnsResponder
        * Service that provides mdns to clients.
        * On windows connects by port 5354 on localhost.
     * mDNSNetMonitor
        * Command line tool that monitors for DNS SD messages.
          * Is an example of using the embedded API.
          * But overwrites some of the embedded API ops:
            * Dispatch to the embedded api message handler doesn't happen, I's kept for it's own use.
    * mdnsNSP
        Provides domian name resolution to windos for *.local domain.
        Does so by registering as winsock namespace provider.
    * NSPtool.
        Registers the mdnsNSP with winsock.
    
* Unknown/Irrelevant in mDNSResponder solution:
    ExplorerPlugin
    ExplorerPluginLocRes
    ExplorerPluginRes
    mDNSResponderDLL
        What does this do?
        Appears to be redundant given mDNSResponder project.
        
 
## Compile 20260104
* mDNSResponder.sln compiles for 64 and 32 bit builds.

## Run:


* Running in windows 11 VM:
    * mdnsresponder runs with errors.
    *dns-sd runs with errors.
    
    * mdnsresponder runs without errors if started with "-server" option.
    * dns-sd can't find server.
        Maybe try registering service?
        Also logs on wind
    * more success:
        In admin command prompt:
            C:\Windows\System32>C:\sw_devel_rob\mDNSResponder.exe -install
            installed service
        In user command prompt:
            C:\sw_devel_rob>C:\sw_devel_rob\dns-sd.exe -V
            Currently running daemon (system service) is version 1661.0.0
            C:\sw_devel_rob>C:\sw_devel_rob\dns-sd.exe  -G v4v6 draytek_2862.local
            DATE: ---Sun 04 Jan 2026---
            20:42:58.224  ...STARTING...
    * Issues:
        1. why do we need to "-install" the service? why doesn't "-server" work?
            Answer: Centennial mismatch.
            If compile dns-sd and dnssd with same settings, won't get this issue/
        2. can't debug the program.
        3. Why and where does mdnsresponder request admin permissions?
            Where: In AppManifest
            Why???
                -install  of service obviously needs perms.
                -server ???? what might this need?
        
## Next steps--done:
* Run locally, and found interesting things...
* Centennial mismatch.
  * I think centennial edition was supposed for mandate UwpAppps.
  * Restricts ports, so opens arbitrary port, then saves loc to env var.
  * The mismatches:
        dnssd  project defines  WIN32_CENTENNIAL
        mdnsResponder does NOT define WIN32_CENTENNIAL
        dns-sd project does not define WIN32_CENTENNIAL
  * 
## Next steps:
* DONE Put this doc into the repo, alongside redame.md  call it RobDevJournal.md
* DONE Migrate repo to be based of my fork.
* Write up issues found migrating repo.
* DONE Build in container:
  * Build successful for all projects of mDNSResponder.sln
* DONE Run in container inside vs2022 to see debug messages.                
* DONE/REVIEW: Fix centennial mismatch.
  * Working with 
* DONE Try disabling run-as-administrator in manifest
  * For this, we want to run command as `mDNSResponder.exe -server` from VC++
  * Then we run non-centennial dns-sd outside, to test.
  * Needed changes in the VC++ linker setting (UAC excution level) **and** in the manifest file
## VM issues:
* Use Bridged network service.
  * If use NAT, can't see any MDNS devices on local net!
* Had to install VS2022
## Outputting debug symbols in vstudio:
* dll must be in same dir as exe to debug.
* must be full debug... not run without debug.



  
## Mofications to remove service:
* How to run without service:
    * Hints that can use embedded service.... but looking at app....
    * Down't want this want to strip-down the MDNS service so
        * Does monitor windows/OS for key events.
            * see: Service.c:::SetupNotifications()
            
            * Most of usd_daemon.c can go away....
                BUT..
                * see:
                    udsserver_handle_configchange()
                        * handles mdns registry config change.
                        * Auto-browse domains... as set in registry.
                        * Auto-register domains.
                * and:
                    udsserver_init()
                        * we definately don't want MOST of this.
                        * we might want the auto domain regsitration stuff.
                        
            * Should check for special handling see:
            
                handle_client_request()
Where is dnsd_clinetshim.c referenced from embedded? 
    Here:
        mDNSEmbeddedAPI.h referenced?
    Missing:
        Seems still need to initialise the API.
        The service does this anyway, and initalsation reqires a lot of work to set everything up.
What to do?
    Suggestion:
        Modify service to:
            1. Be simple exe, without any service features:
                Entry point is given, that overloads the m_dns.c initalsiation ops.
            2. Disable the creation of the connction to client.
                Drop udsserver_init()
                Check if we need to do anything to initialise for our single 'client'
            3. All other client work done via dnsd_clinetshim.c
        
    What would we modify:
    
        ServiceSpecificInitialize()
        
            Remove udsserver_init()
            
                ... possibly replace with someting else, as the client connection initialises some things.
                
                5175 onwards looks like thing we keep.
                
            uds_socket_setup()
            
                Nothing to keep here....
                
                But invesitigate: connect_callback()
                
                    Nothing to keep here, but...
                
                    look at udsSupportAddFDToEventLoop()
                        If need to keep parts of this consider that request preameter is initiased here.
                        
        udsserver_init()
        
            5175 onwards....
            
                Monitors browse domains for change.
                
                    Calls:
                        
                        add_domain_to_browser()
                        
                            Potentially modifying existing browse domains.
                
Maybe we run with sockets instead?
    Just change a macro.... see  udsserver_init()  and follow down till FDs are set...
        there's a macro to select sockets or named pipes.
        If we use named pipe can work well.
    In addition:
        * Don't advertise anything.
            See:  ServiceSpecificInitialize()
        * Don't read registry.
            See:  kServiceParametersNode and relatedf.
        * Don't do unicast.
            set the UNICAST_DISABLED in compiler options.
            See also: CanReceiveUnicast()

    https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/
    AF_LOCAL is supported.
                 
    For managing this would use:
    
        %APPDATA%/russetrob/bonjour/<Pid>/pipe
        
        Use:  ExpandEnvironmentStringsForUser function.
        
        Better: ExpandEnvironmentStrings  does this for current user.
        
    Also, we can do:
        https://learn.microsoft.com/en-us/windows/win32/procthread/creating-a-child-process-with-redirected-input-and-output
        
        This shows how to...
            * Inovke a process and have stdin/stdout avaialable.
            
    And finally, to clean up the chidren:
        https://stackoverflow.com/questions/6259055/createprocess-such-that-child-process-is-killed-when-parent-is-killed
        https://stackoverflow.com/questions/3342941/kill-child-process-when-parent-process-is-killed
        
        * Create job process.
        * Add child process to job.
        * Mark job such that closing job handle terminates process.

     To close proces cleanly, send message on stdin to the process.
     
Process events on stdin:
* in the following:
  * `>` = parent to child.
  * `<` = child to parent
  * << ....> = some event out-of-band vs. stdio for parent-child comms.
```
    > NewNamedPipe
    < NamedPipe = <Full path to named pipe> // we use globally unique name for named pipe
    > Exit
    <<Wait for GetExitCodeProcess != STILL_ACTIVE,  or WaitForSingleObject on the process handle>    
        Note: Accrdoing to: https://github.com/haskell/process/issues/77
        This is not suffcient for all cases, but shouldn't be relevant to us.
        Registing the child process as a job process should be sufficient for our needs.
```        
Open child process without window:
    https://stackoverflow.com/questions/7063859/c-popen-command-without-console
    
### Issues seen when running mDNSResponder without admin.
* No issues found so far reqiring admin access.
* Also found a free-null reference error... fixed.
```
[mDNSWin32] platform init
[mDNSWin32] Unicast UDP responses *not allowed*
[mDNSWin32] HIHardware: Windows
[mDNSWin32] setting up socket 0.0.0.0:52428
[mDNSWin32] setting up socket [0000:0000:0000:0000:0000:0000:0000:0000%0]:52428

[ASSERT] error:  10022 (An invalid argument was supplied.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 3079, "SetupSocket"

[mDNSWin32] platform init done (err=0 no error)

[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 1810, "SetSearchDomainList"


[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 2029, "SetDomainFromDHCP"

[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff::
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.255.255.0
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.0.0.0
Running service directly
[mDNSWin32] setting up interface list
[mDNSWin32] tearing down interface list
[mDNSWin32] tearing down interface list done
[mDNSWin32] nice name "DESKTOP-C340V5O"
[mDNSWin32] netbios name "DESKTOP-C340V5O"
[mDNSWin32] netbios domain/workgroup "WORKGROUP"
[mDNSWin32] host name "DESKTOP-C340V5O"
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff::
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.255.255.0
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.0.0.0
[mDNSWin32] Interface   {EC68D501-E938-4280-A191-CAFF5B2509B8} (0x00000012) 192.168.1.21:0
[mDNSWin32] setting up interface
[mDNSWin32] setting up socket 192.168.1.21:0
[mDNSWin32] Registered interface 192.168.1.21:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] Interface   {EC68D501-E938-4280-A191-CAFF5B2509B8} (0x00989692) [2001:08B0:FFBF:0001:0000:0000:0000:0797%0]:0
[mDNSWin32] setting up interface
[mDNSWin32] Registered interface [2001:08B0:FFBF:0001:0000:0000:0000:0797%0]:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] Interface   {EC68D501-E938-4280-A191-CAFF5B2509B8} (0x00989692) [FE80:0000:0000:0000:93D6:B7BA:DE32:DB5D%18]:0
[mDNSWin32] setting up interface
[mDNSWin32] Registered interface [FE80:0000:0000:0000:93D6:B7BA:DE32:DB5D%18]:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] Interface   {91D13558-29F7-11EB-ABA3-806E6F6E6963} (0x00989681) [0000:0000:0000:0000:0000:0000:0000:0001%0]:0
[mDNSWin32] setting up interface
[mDNSWin32] setting up socket [0000:0000:0000:0000:0000:0000:0000:0001%0]:0

[ASSERT] error:  10022 (An invalid argument was supplied.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 3079, "SetupSocket"

[mDNSWin32] Registered interface [0000:0000:0000:0000:0000:0000:0000:0001%0]:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] setting up interface list done (err=0 no error)

[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 1810, "SetSearchDomainList"


[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 2029, "SetDomainFromDHCP"

[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff::
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.255.255.0
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.0.0.0
[mDNSWin32] session closed
[mDNSWin32] session closed
[mDNSWin32] session closed
```
* With administrator prompt:
  * Only issues are the inMem meaages, as we haven't applied this patch to the other branch.
```
[mDNSWin32] platform init
[mDNSWin32] Unicast UDP responses *not allowed*
[mDNSWin32] HIHardware: Windows
[mDNSWin32] setting up socket 0.0.0.0:52428
[mDNSWin32] setting up socket [0000:0000:0000:0000:0000:0000:0000:0000%0]:52428

[ASSERT] error:  10022 (An invalid argument was supplied.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 3077, "SetupSocket"

[mDNSWin32] platform init done (err=0 no error)

[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 1809, "SetSearchDomainList"


[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 2027, "SetDomainFromDHCP"

[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff::
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.255.255.0
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.0.0.0

[ASSERT] assert: "inMem"
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 754, "mDNSPlatformMemFree"


[ASSERT] assert: "inMem"
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 754, "mDNSPlatformMemFree"

Running service directly
[mDNSWin32] setting up interface list
[mDNSWin32] tearing down interface list
[mDNSWin32] tearing down interface list done
[mDNSWin32] nice name "DESKTOP-C340V5O"
[mDNSWin32] netbios name "DESKTOP-C340V5O"
[mDNSWin32] netbios domain/workgroup "WORKGROUP"
[mDNSWin32] host name "DESKTOP-C340V5O"
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff::
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.255.255.0
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.0.0.0
[mDNSWin32] Interface   {EC68D501-E938-4280-A191-CAFF5B2509B8} (0x00000012) 192.168.1.21:0
[mDNSWin32] setting up interface
[mDNSWin32] setting up socket 192.168.1.21:0
[mDNSWin32] Registered interface 192.168.1.21:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] Interface   {EC68D501-E938-4280-A191-CAFF5B2509B8} (0x00989692) [2001:08B0:FFBF:0001:0000:0000:0000:0797%0]:0
[mDNSWin32] setting up interface
[mDNSWin32] Registered interface [2001:08B0:FFBF:0001:0000:0000:0000:0797%0]:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] Interface   {EC68D501-E938-4280-A191-CAFF5B2509B8} (0x00989692) [FE80:0000:0000:0000:93D6:B7BA:DE32:DB5D%18]:0
[mDNSWin32] setting up interface
[mDNSWin32] Registered interface [FE80:0000:0000:0000:93D6:B7BA:DE32:DB5D%18]:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] Interface   {91D13558-29F7-11EB-ABA3-806E6F6E6963} (0x00989681) [0000:0000:0000:0000:0000:0000:0000:0001%0]:0
[mDNSWin32] setting up interface
[mDNSWin32] setting up socket [0000:0000:0000:0000:0000:0000:0000:0001%0]:0

[ASSERT] error:  10022 (An invalid argument was supplied.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 3077, "SetupSocket"

[mDNSWin32] Registered interface [0000:0000:0000:0000:0000:0000:0000:0001%0]:0 with mDNS
[mDNSWin32] setting up interface done (err=0 no error)
[mDNSWin32] setting up interface list done (err=0 no error)

[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 1809, "SetSearchDomainList"


[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 2027, "SetDomainFromDHCP"

[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff::
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.255.255.0
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.0.0.0
[mDNSWin32] TCP/IP config has changed

[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 1809, "SetSearchDomainList"


[ASSERT] error:  2 (The system cannot find the file specified.)
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 2027, "SetDomainFromDHCP"

[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff::
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.255.255.0
[mDNSWin32] getifaddrs_ipv6: IPv6 mask = ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
[mDNSWin32] getifaddrs_ipv6: IPv4 mask = 255.0.0.0

[ASSERT] assert: "inMem"
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 754, "mDNSPlatformMemFree"


[ASSERT] assert: "inMem"
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 754, "mDNSPlatformMemFree"


[ASSERT] assert: "inMem"
[ASSERT] where:  "C:\sw_devel_rob\mDNSResponder\mDNSWindows\mDNSWin32.c", line 754, "mDNSPlatformMemFree"

```