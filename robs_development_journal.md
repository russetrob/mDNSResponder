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
        * This is the client-side implementaton of the bojour API.
    * dns-sd
        * Command line tool
    * DLLX
        * Appears to be a COM interface to the dnssd library
    * mDnsResponder
        * Sevice that provide
     * mDNSNetModitor
        Command line tool that monitors for DNS SD messages.
        * Is an example of using the embedded API.
        * But overwrites some of the enbedded API ops:
            * Dispatch to the embedded api mesage handler doesn't happen, I's kept for it's own use.
    * mdnsNSP
        Provides domian name resolution to windos for *.local domain.
        Does so by registerting as winsock namespace provider.
    * NSPtool.
        Resiters the mdnsNSP with winsock.
    
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
    
    * mdnsresponder runs without errors if started with "-server" oprion.
    * dns-sd can't find server.
        Maybe try registering service?
        Also logs on wind
    * more sucess:
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
  * I think centennial edittion was supposed for mandate UwpAppps.
  * Reistricts ports, so opens arbirary port, then saves loc to env var.
  * The mismatchs:
        dnssd  project defines  WIN32_CENTENNIAL
        mdnsResponder does NOT define WIN32_CENTENNIAL
        dns-sd project does not define WIN32_CENTENNIAL
  * 
## Next steps:
* Put this doc into the repo, alongside redame.md  call it RobDevJournal.md
* Migrate repo to be based of my fork.
* Run in container inside vs2022 to see debug messages.                
* Fix centennial mismatch.
* Try disabling run-as-administrator in manifest
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
  * << ....> = some event out-of-band from stdio for parent-child comms.
```
    > NewNamedPipe
    < NamedPipe = <Full path to named pipe> // we use globally unique name for named pipe
    > Exit
    <<Wait for GetExitCodeProcess != STILL_ACTIVE,  or WaitForSingleObject on the process handle>    
        Note: Accrdoing to: https://github.com/haskell/process/issues/77
        This is not suffcient for all cases, but shouldn't be relevant to us.
        Registing the child process as a job process should be sufficient for our needs.
```        
        