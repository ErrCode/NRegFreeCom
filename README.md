## NRegFreeCom

 Load native libraries, call native functions, do COM objects without registration, do COM registrations in runtime, in .NET

 COM is good for native interop and easy IPC, but lacks clean coded way to do such interaction without deployment/compile time registry in .NET.

 Managed code is XCOPY on 32/64. Making native interlop also XCOPY by using pattern of deploying both versions of native libs and deciding in runtime, like OS libraries PInvokes work.
 Managed code have AppDomains with dll search and memory isolation, striving to do the same of native COM.

 This project contains samples and reusable patters of things above.

## Content
* PInvokes used to work with native libraries and reg free COM objects
* Sample how to load native library with module definition as assembly and providing it with managed COM services
* Sample of native library implementing managed COM interface searched and  instatiated by managed code using manifest
* Managed out of process COM server which uses Running Object Table to broadcast it instance (zero Windows Registy info)
* Managed out of process COM server which does registration at start and unregisters when closing
* Working with native library dependencies and resources

## Build and deployment

* `NRegFreeCom` is redistributable managed library, can be build in Release and Debug mode ShardDevelop or Visual Studio 2010 SP1. Is build for `Any Cpu`
* Library testing is done on XP SP3 and Windows 7 SP2
* Run `deploy.bat`

### Samples build

* Visual Studio 2010 SP1 (needs Visual C++)
* Debug|Mixed Platforms
* Windows 7 SP 2


## Samples

```csharp

       [Test]
		[Description("Creates and calls managed registration free object")]
        public void CreateInstanceWithManifest_inProcessManagedServer_OK()
        {		    
            var path = Path.Combine(Environment.CurrentDirectory, @"RegFreeCom.Implementations.dll.manifest");
            var guid = new Guid(RegFreeComIds.CLSID);

            var obj = ActivationContext.CreateInstanceWithManifest(guid, path);
            var inf = (IRegFreeCom)obj;
            var result = inf.Answer();
			
            Assert.IsTrue(result == 42);
        }
		
		[Test]
        [Description("Loads library from subfolder with dependencies searched in this subfolder")]
        public void TestDllDependenciesLoading()
        {
            var loader = new AssemblySystem();

            // C# loads C++ from Win32 or x64 subfolder
            var anyCpu = loader.GetAnyCpuPath(loader.BaseDirectory);
            loader.AddSearchPath(anyCpu);
            var module = loader.LoadFrom(anyCpu, "NativeLibraryConsumer.dll");
            var fn = module.GetDelegate<fnNativeLibraryConsumer>();
            object[] retval;

            Assert.IsTrue(42 == fn(out retval));
            loader.Dispose();
        }
		
        [Test]
        [Description("Tests basic workflow of custom Dispatcher")]
        public void GetAndRunDispatcher_invokeAndShutdown_invocationInDispatcherDone()
        {
            //start custom Dispatcher for Windows Message Pump
            NRegFreeCom.IDispatcher disp = null;
            var created = new ManualResetEvent(false);
            var t = new Thread(x =>
            {
                disp = NRegFreeCom.Dispatcher.CurrentDispatcher;
                created.Set();
                NRegFreeCom.Dispatcher.Run();
            });
            t.SetApartmentState(ApartmentState.STA);
            t.Start();
            created.WaitOne();

            // run custom Dispatcher
            int threadId = -1;
            bool wasAct = false;
            Action act = () =>
            {
                wasAct = true;
                threadId = Thread.CurrentThread.ManagedThreadId;
            };
            disp.Invoke(act);
            disp.InvokeShutdown();

            // invocation in Dispatcher thread was done
            Assert.AreEqual(threadId, t.ManagedThreadId);
            Assert.That(wasAct, Is.True);
        }

    [TestFixture]
    public class RegAsmTests
    {
        [Test]
        public void User_RegisterInProcSever()
        {
            var type = typeof (RuntimeRegServer);
            RegAsm.User.RegisterInProcSever(type);
        }

        [Test]
        public void User_UnregisterInProcSever()
        {
            var type = typeof(RuntimeRegServer);
            RegAsm.User.UnregisterInProcSever(type);
        }
    }

	        [Test]
        public void Create_oneKeyWithOneValue_OK()
        {
            var text = 
@"Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SOFTWARE\MyKey]
" +
"\"MyValue\"=\"_RegFreeComRotClass\"";
            var stream = stringToStream(text,Encoding.UTF8);
            var reader = new RegFileReader(stream);
            var key = reader.RegValues[@"HKEY_LOCAL_MACHINE\SOFTWARE\MyKey"];
            Assert.IsNotNull(key);
            Assert.IsTrue(key.ContainsKey("MyValue"));
        }
```

See tests and samples in code for other functional (like inter process communication without server Windows registry entry; loading and initializing native libraries, methods and COM objects).

##Notes

### In order to ease integration of C++ and C# I think could be good to:

* C libs loaded to provide HRESULT STDAPICALLTYPE exports with COM memory management methods used to be similar to COM lifecylte functions exported for uniformity of calls. 
* Strive to have expericene of new WinRT (*.winmd, C++/CX, WRL). No Windows registy involved. XCOPY deployment. 
* COM interfaces to be as simple as possible so these could be implemented manually (without wizards) by developer with low C++ skill.



## Other semi automatic approaches of doing native interop

* IDL COM ATL Wizards, tlbimp
* https://github.com/mono/CppSharp
* http://clrinterop.codeplex.com
* C++/CLI
* C++/CX, WRL
* SWIG
* CXXI (Linux, gcc)


## TODO:
* Enumerate DLL exports
* Make custom resources sample and read this.
* Try to do with resources what WinRT did. Embeed CLI metadata describing reg free COM component, generate manifest from it, and create COM, use Metadata Interfaces. http://msdn.microsoft.com/en-us/library/ms233411.aspx.
* Make delegates for all standards DEFs( DLL, COM)
* Fake registration out of process ROT client instead of using proxy
* Add dependency conflicts for managed and for native and resolve both by SxS manifests tests
* Imitate AppDomains based on runtime binding
* Add PE code (detecting managed headers, DEF and COM headers).
* Research how  api-ms-win-core can be employed for isolation http://www.nirsoft.net/articles/windows_7_kernel_architecture_changes.html
* User free tools. Use SharpDevelop to build all C#. Use NMAKE/CL/MIDL and/or C++ template of SD to build all C++. Compile C++/C# in runtime and start processes to make tests more robust.
* Investigate WiX and Import/Export reg to Xml projects which contain reg parsing and to XML code, drop support here or reuse then

### Isolation and integration

  Native code does not have AppDomains and have static linking for isolation. There are technologies which helps to solve/mitigate problems of conflicting components while dynamic linking of native components:
* Windows provided features like Activation Contexts, application `manifest`s, `dll.2.config`, dynamic and controlled libraries loading and symbols binding, SxS
* EasyHook - The reinvention of Windows API Hooking https://easyhook.codeplex.com/
  
  Isolation can be achieved by design:
* Multiprocess architecture
* There are Dependency Injection and Inversion of Control tools for C++, but because of lack of runtime metadata in native code by default not so powerful as in .NET

This lib strives to make .NET engine to load native code in isolated manner.

## What makes C# .NET ready for developing Windows deskops apps

Problem
---
.NET desktop applications take long time to start, are slow and CPU consuming while using runtime features like reflection, are slow and CPU consuming when integrating with native code (e.g. can call only high level COM API while native code can call hich performance API of native components).

Context
---
- User can open and close desktop application many times.
- Desktop application can be deployed and run in restricted environment, hence cannot put assemblies into GAC.
- Windows is unmanaged OS with all core user apps written in unmanaged code, hence .NET need to be well integrated into these to call API with high performance.
-  Main .NET language - C# - does not have compile time meta programming build in.
-  .NET has many places with runtime code generations which should avoided.

Enforces
---
Using 3rd party tools for compile time code degeneration instead of runtime. Loose will be mostly in ease of build and support. But this will allow .NET be good citizen e.g. as MS Office add-in or as applications which does auto start after window launch.

For now
---
- Parallel Just in Time(JIT) compilation and good Ahead of Time(AOT) support (NGEN). .NET 4.5 has both.
- Low overhead COM events support via ComEventsHelper. .NET 4.0 has it.
- More COM support like ICustomQueryInterface since .NET 4.0. Looks almost each new version of .NET adds more stuff.
- Fast first usage of XML serializes by default, faster start up time. .NET 4.5 has it.
- Coexsitence of big C++ and big C# heap in the same process. Looks almost each new version  of .NET adds more APIs for GC and CLR hosting tuning.
- Enough of free and opensource tools for .NET profiling.
- Opensource tools and libraries of COM and C interop [2] [3]

Better to have in future
---
- Widespread of .NET 4.5.
- Integration of COM with CLI metadata. WinRT apps has some. Need the same for Desktop.
- Compile time generics-templates, good tool for AOP or other kind of metaprogramming.
- Compile time Dependency Injection container.
- Compile time analysis of runtime reflection and code generation usage to prevent both.
- Support for fast interface/attribute based inter process communication(IPC) with pregeneration of proxy/stub like in [1]
- Some way to do RAII to release references counted objects (specifically STA COM) instead of calling Marshal.Release or wrapping into using directly.

[1] http://visualstudio.uservoice.com/forums/121579-visual-studio/suggestions/4034029-add-support-of-ms-rpc-into-c-generate-c-from-i
[2] https://clrinterop.codeplex.com/


##Q&A

Q: If I want to load some DLLs of which are there inside "Program" folder, using Regfree way?
A:
If just dll then 
```csharp
            var program = Path.GetFullPath("Program");
            var dlls = new NRegFreeCom.AssemblySystem();
            dlls.AddSearchPath(program);
            var dll = dlls.LoadFrom(Path.Combine(program, "MyDll.dll"));
```
If want COM object 
```csharp
            var manifest = Path.Combine(program, "MyDll.dll.manifest");
            NRegFreeCom.ActivationContext.UsingManifestDo(manifest,
                                                          () =>
                                                              {
                                                                  IMyCom comObj = new MyCom();// COM object with manifest
                                                              });
```
COM will load all dll dependencies mentioned in manifest automatically if these are located in the same folder (and with manifests if have).