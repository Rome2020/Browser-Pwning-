# Browser-Pwning
A proper well structured documentation for getting started with chrome pwning &amp; v8 pwning 


# Structure of document
| How this doc is organised  | 
| ------------- | 
| 1. Motivation | 
| 2. Actual Study Material | 


# Motivation

Browsers are one of today’s most used pieces of technology. On every off the shelf computer, if we just plug and play we will see a browser installed. That’s why from an attacker’s perspective and a threat model perspective it’s very rewaradable if the attacker is able to compromise the browser through a malicious page. Given the upper arguments I choose to study Google’s Javascript Engine, particularly v8. 

Given the gigantic nature of v8’s project,I choose as a starting point the interpreter, namely d8.Although extensive research has been done already on d8, we hope to at least find one bug and if not be able to move forward with research on browser exploitation since v8 provides entry ground for the basic exploit development strategies used on browser exploitation .

 Another reason why I choose v8 as a target is that it’s used across multiple browsers. 
 
 If we look under the hood we can see that the engine is used in MicrosoftEdge too, so there is a multiple bug bounty award chance. For the underlying operating system   the researcher will be using a mix of windows and linux since there’s no restriction in terms of obtaining a shell since v8 bugs allow code execution through wasm pages and it’s not bound to any platform specific. 
 
 
Unfortunately, while an exploited bug in v8 would result in code execution, we won’t be able to execute any code due to the sandbox, and as such we would get code execution in the context of the renderer,which would not allow us to execute code on the machine. For that we would need another exploit for the sandbox,thus we would need a full chain in order to exploit the system.

And such we define the following goals in order to be able to get started with browser hacking

 | Goals  | 
| ------------- | 
| Learn,map,master the internal structures used by V8 engine and the key points of Chrome architecture | 
| Learn and master the basics procedures necessary for browser exploitation  | 

# Learn,map,master the internal structures used by V8 engine and the key points of Chrome architecture 
  At the first stage of the project it's necessary to gather as much knowledge regarding Chrome Architecture and how each componenet interact with each other. In order to better understand this we need to the Chromium Project into multiple subcomponenets such that we are able to isolate everything and analyse it properly. More precisely in how many subcomponents each of the following components breaks 
  V8 Architecture
  Chromium Architecture
  Blink Architecture
  

Now the most logical step for a first step is understanding chromium architecture.
   So ok we want to exploit the browser, but what happends when we first start the browser? Well after clicking the chromium executable , the executable starts some processes.
   The order and their name is as follows:
   
     First one is called content process. What does this process do ?
          * It is know as the startup process.
          * it is also the main process
          * it's responsable for starting the actuall main process which is called browser process 

          
          Now that we briefly know what it does it is time to go more indepth on it:
            *So the way Chromium works on Windows at least, is that it compiles the files into a dll , and after that it loads it into memory. So the core logic of Chroium browser is in chromium.dll. 
            This is also confirmed by the code
             ![2](https://user-images.githubusercontent.com/25670930/146623873-a9385ba3-3836-4d96-817e-c66d8f7bfc64.PNG). This is take from chrome_exe_main_win.cc and if u are currious to read the whole code it's located at chromium/src/chrome/app . Ok, going down with the execution we can see it calls MakeMainDllLoader() to call the dll loaded class after that it launches the "loader" aka it loads the chrome.dll and if it is necessary restarts it with the necessary command lines. To further analyse the Loader we need to understand it's code, which in the same directory, in the file mail_dll_loader_win.cc .Scrolling all the way down of the file we can see the call to MakeMainDllLoader which in terms calls based on the verion you have ChromeDllLoader or ChromiumDllLoader. ![3](https://user-images.githubusercontent.com/25670930/146624258-cbb87c7e-e48c-4fbf-9280-a390a5b37010.PNG) We can see that ChromiumDllLoader is a class which inherites from MainDllLoader. We can see from the definition that the class simply does is to load the dll based on the arguments passed to cmdline and process type![6](https://user-images.githubusercontent.com/25670930/146624860-99c0fb44-2ec4-4323-bc4b-e6849e423179.PNG). Analysing the Launch method we can understand some things that happen before chrome.dll starts: We can understand from this comment that "// Launching is a matter of loading the right dll and calling the entry point. // Derived classes can add custom code in the OnBeforeLaunch callback." launching chrome is a matter of loading a bunch of dll which actually do the work. Secondly it takes the arguments passed to command line and further it initialises the sandbox services.![q](https://user-images.githubusercontent.com/25670930/146625301-e8aea9cf-4223-4487-b5f2-e7da492a75ca.PNG). First it checks to see if it's the browser who's calling the sandbox initialisation than it checks if the process which called the initialisation of the sandbox was called as a cloud print service. It also checks if  --no-sandbox was passed to the binary,which basically tells to the binary not to run in a sandbox. it checks if wether any of these were set and if either is true it calls the sanbox with the respective options. than in the end we get to ![8](https://user-images.githubusercontent.com/25670930/146625574-0d1a67c4-c0f8-4bbc-bed6-631bad59b5f8.PNG) which is what we are interested in, this representing the wrapper to calling chrome_main.
             *Now lets see what it does. we open chromium.dll in binja 
             Based on this picture from https://blogs.igalia.com/jaragunde/files/2019/03/chrome-init-sequence.png we have a rough ideea of what should we look after in binja
             This is how the graph looks in binja![research](https://user-images.githubusercontent.com/25670930/146561718-139d5a5a-4759-4a44-98c2-2efabee4cb3d.PNG) . To be able to better track it down look for a function called ChromeMain. This is chrome's main(aka core) logic.Inside of it we can see the afro-mention phases of how chrome.dll runs at startup:
             Here we can see the first it calls some functions such as sub_180001420,sub_180017020,sub_180017020 which do some checking to see some details regarding how chrome was installed/compiled. Using the source code we can track their atributes. First function,sub_180001420 corresponds to UmaHistogramEnumeration,next we have sub_180017020 which is InitializeFromPrimaryModule and third and last one sub_180017020 is chrome_main_delegate.
             We keep repeating chrome_main_delegate but we never defined what it's purpuse is. ChromeMainDelegate is a class which inherits from ContentMainDelegate and mainly provides startup related functions and process call processing. !In case you want you can implement a custom ContentMainDelegate interface to change the default behavior of the Content module, and use a custom ChromeMainDelegate class in Chromium to customize the behavior of the startup process. 
             
              ![Capture](https://user-images.githubusercontent.com/25670930/146572896-6d40dbf2-fcec-4a6a-a0eb-13a4f26aa7df.PNG). Inside of it there are some function calls which do some xor on some data regions andconcat it with some regitry to get some options regarding chrome startup ![Capture](https://user-images.githubusercontent.com/25670930/146581324-fdb4098f-fe78-49d4-b2a4-f7aab5c36be2.PNG). Also note that function sub_180001510 is creating a scopted pointer reference with two callbacks.Not really important thing that it does but it's interesting to learn what a BindStateBase  is. We will meet it a lot in the chrome's basecode.
              Now you may be wondering what do the last three lines, precisly ![Capture9](https://user-images.githubusercontent.com/25670930/146587641-6df35a9f-4d55-4f82-912a-6a3af09900a4.PNG) do. so first line basically creates a std::unique_ptr<> for Closures. WTF is a closure!? well if it were to cyte from mozila dev" closure is the combination of a function bundled together (enclosed) with references to its surrounding state (the lexical environment). In other words, a closure gives you access to an outer function’s scope from an inner function. In JavaScript, closures are created every time a function is created, at function creation time." It if were to use human language it's a function whiting a function which acceses a variable withing the outher function. Eg![1](https://user-images.githubusercontent.com/25670930/146588196-84634271-c7b4-48c9-ba40-a54b92e7f057.PNG) . So in essnce it makes shure that the closure will execute. and the other two lines just set some specific behavior for dumps when chrome crashes.InstallDetails::Get().VersionMismatch() patchuie aici
              Going futher it checks for it's version at runtime and in case it doesn't match it it crashes and we arrive at the command line parsing ![commandline](https://user-images.githubusercontent.com/25670930/146590951-0043de30-11ca-448a-8928-1a7268d21696.PNG)
              Stepping inside sub_184171800 we can see it is not that big ![commandline2](https://user-images.githubusercontent.com/25670930/146591178-c44dfc19-097b-4fa3-8e64-26c0c131a324.PNG) First it takes the arguments passed to the current process, than calls a function which takes a StringPiece basically a class wrapper for std::string but a little bit cooler and that function basically checks if the binary was called headless,compares is the name of the binary which was run is chrome , checks if USE_HEADLESS_CHROME is set and than goes further into next step which is content main.
              Based on the arguments provided , content main's job is to start the respective shell.
              Now in case we don't provide any arguments to the shell the execution flow goes to ![10](https://user-images.githubusercontent.com/25670930/146630552-d76a440c-c4ae-4673-92ce-bcb7e76e52bc.PNG). In order to understand what it does we need to verify to source of it. We can find it at /src/content/app/content_main.cc
              We will have to scroll all the way to the bottom and and there we will find the definition of the ContentMain function. There we see it calls two functions. One which initialises a ContentMainRunner which is the class that handles all the "process creation" in the context of creating the browser ipc sqli net and the rest. And the other one which basically check the type of subprocesses and feed it to the ContentMainRunner class. 
              Now let's take a step back in order to understand the codeflow. First let's examine the RunContentProcess as ContentMainRunner is quite complex.First step it does is to create a GlobalActivityTracker. Omg !!! you guess it google tracks you :))) Nah just kidding pls don't sue me Google. But on a serious note this track each thread which is a thread tracker and it does a couple of cute things for better debugging such as:![Capture](https://user-images.githubusercontent.com/25670930/146662120-10749d1b-041c-45c3-87d6-deff628da350.PNG) which in essence means that there will be a unique integer used as an identifier for each thread to better understand what cause the process to break.
              eg: ```
                  kTypeIdActivityTracker = 0x5D7381AF + 4,   // SHA1(ActivityTracker) v4
                  kTypeIdUserDataRecord = 0x615EDDD7 + 3,    // SHA1(UserDataRecord) v3
                  kTypeIdGlobalLogMessage = 0x4CF434F9 + 1,  // SHA1(GlobalLogMessage) v1
                  kTypeIdProcessDataRecord = kTypeIdUserDataRecord + 0x100,
                  ```
              tracking a process life stage which is defined as 
              ```
                  // The phases are generic and may have meaning to the tracker.
              PROCESS_PHASE_UNKNOWN = 0,
              PROCESS_LAUNCHED = 1,
              PROCESS_LAUNCH_FAILED = 2,
              PROCESS_EXITED_CLEANLY = 10,
              PROCESS_EXITED_WITH_CODE = 11,

              // Add here whatever is useful for analysis.
              PROCESS_SHUTDOWN_STARTED = 100,
              PROCESS_MAIN_LOOP_STARTED = 101,
              ```
              log asap when a process exits,save the information regarding the module aka when when a module(aka a componenet of chromium) is loaded and basically the same functinolity repeated but different for different threads and classes. In case you are interested you can find it at chromium/src/base/debug/activity_tracker.h 
              Afterwards checks if "Main()" was called before. Here i think they mean if the ChromeMain() was called before. They basically check if there were passed any command to the content process was called with any arguments.And in case it was , they make sure the browser process will  get the same args.They than check for platform specific and in case windows is found the initialise they own handler by doing CreateATLModuleIfNeeded, than it's the same story checking for platform specifics and and they pass the arguments using SetupCRT and we get to the part where we initialise the ipc so we can talk to the other processes after we have spawned them. Here is the code responsabile for that. We won't go into details about it , as we will come back later discussing more indepth about the ipc mechanism of chrome. Just know this for the moment, this is the mechanism which facilitate the talking between the mutli architecture processes of chrome. And in case you don't know for what stands IPC, it's for inter process communication.![mojo](https://user-images.githubusercontent.com/25670930/146662673-32045099-b6b5-4a6c-89ff-5dd6efc4297b.PNG). Than what it does it "fordwards" rather more than sets the arguments for ui using ui::RegisterPathProvider, call the tracker and gets the result using content_main_runner->Initialize(std::move(params)) ,create a parent cosole, which i guess what they mean is that they create a parent process does some more checking and than we get to the important part which is ![Capture2](https://user-images.githubusercontent.com/25670930/146662970-9d31f08d-2d66-4b71-b344-650ded989893.PNG) . Here we see a call to a function called IsSubprocess. ![Capture3](https://user-images.githubusercontent.com/25670930/146663018-b27e2f7e-b8fe-4c48-b4fe-c3b2e7ee75c7.PNG) what it does basically to avoid the old boilderplate code it checks in one function the type of the process it receives on command line  and in case an option was passed it chooses from the following it creates the respective process. 
         ```return type == switches::kGpuProcess ||
         type == switches::kPpapiPluginProcess ||
         type == switches::kRendererProcess ||
         type == switches::kUtilityProcess || type == switches::kZygoteProcess;```
        From there we get to content_main_runner->Run(); which in essence does all the magic. Let's examine it under the hood. So content_main_runner is of class ContentMainRunner duhh.. which can be found at content\app\content_main_runner_impl.cc
         To actually be able to understand the code flow for this we have to take it from the bottom to the top. And this being said , we scroll down again and we find that actually ContentMainRunner::Create() calls infact ContentMainRunnerImpl::Create(); which makes us realise that in fact we will have to search for ContentMainRunnerImpl definition not for ContentMainRunner, which we find in content_main_runner_impl.h in the same already mentioned folder.![Capture 4PNG](https://user-images.githubusercontent.com/25670930/146663921-1188df62-5aca-4fad-a081-3c338380d3f4.PNG)
We see that it inherits from ContentMainRunner and we can see it's core methods. Really there's not too much here as it's behaviour is mostly overwritten in content_main_runner_impl.cc
         Inside content_main_runner_impl.cc, inside run method it first does some checking using DCHECK(what the hell is this function?(```The CHECK() macro will cause an immediate crash if its condition is not met. DCHECK() is like CHECK() but is only compiled in when DCHECK_IS_ON is true (debug builds and some bot configurations, but not end-user builds).``` quotes from google docs :) ) to see if is_initialized,content_main_params_,is_shutdown_ are set.![Capture6](https://user-images.githubusercontent.com/25670930/146664265-9ba18dca-8f2b-46d2-b3a2-82583a9378c3.PNG) Afterwards is gets the arguments and determines the type from which i have mentioned earlier![Capture7](https://user-images.githubusercontent.com/25670930/146664279-d5b2dfbd-c293-4a07-b5d1-22306c592dfc.PNG) Than in case we can't find that we call InitializeFieldTrialAndFeatureList()  delegate_->PostFieldTrialInitialization(); and post is as message using mojo.![Capture8](https://user-images.githubusercontent.com/25670930/146664315-a9cccbec-58ce-46ce-8c4f-603cf1fc697f.PNG) . Next what it does is to sets some stuff for ui![Capturze](https://user-images.githubusercontent.com/25670930/146704713-43f8cdb7-faa3-4aac-b8a8-e4ea24cf6e6f.PNG) again based on what it had been passed on command line and calls RegisterMainThreadFactories which is a wrapper for  RegisterUtilityMainThreadFactory which only sets g_utility_main_thread_factory  which by the name i think it creates the main thread which is like the thread which is the watcher for the rest of the threads and finally launches the browser process. ![Captu1re](https://user-images.githubusercontent.com/25670930/146705128-256e58f3-936f-4650-bef1-56dc22e557e2.PNG)
         
         In case you don't trust me here is chromium disassambled in binja and we will also see some in memory analysis of that 
            Lucky us, we have chome.exe.pdb which allows us to have debug symbols and we won't have any pain while reversing the binary
             Since we work on chrome on windows, the main entry for the program is wWinMain:
             Here is how the graph looks
              ![Captur3e](https://user-images.githubusercontent.com/25670930/146705695-32e3ee52-ddfe-47ae-abd6-9330d0a5ca32.PNG). The function itself is heudge and such i wil only show the necessary parts of it. After a bunch of initialisations we get to a point where it calls MakeMainDllLoader()![Capture4](https://user-images.githubusercontent.com/25670930/146706065-78949ea9-4c61-40dd-bc4b-69edff78a0e7.PNG), which we have explained what it does. Right so how do we dynamically debug chrome and prove everything i have already mentioned ? first we load it in windbg, and than lm and look for the name of executable.After that we look for a function called chrome!MakeMainDllLoader, we set a bp and let the execution go.![Captu2re](https://user-images.githubusercontent.com/25670930/146706210-29427753-5780-4952-ba6e-10d9538a95c6.PNG). we get inside of it ![Captu11re](https://user-images.githubusercontent.com/25670930/146706268-cb168de2-46e0-4d20-9a5a-8a374efe8420.PNG) let it run until it hits ret and we get out of it we we get into chrome!wWinMain+0x764 ![Captur33e](https://user-images.githubusercontent.com/25670930/146706345-08abf116-b834-4920-99f0-f0bfb36a0671.PNG). we than let it run until chrome!MainDllLoader::Launch and step into it. From there we bp at chrome!MainDllLoader::Load so we can see how chrome.dll is loaded into memory. And as we can see among the first things which was loade was chrome.dll ![Captu123re](https://user-images.githubusercontent.com/25670930/146706725-ab63fd49-90ca-4bbd-8c33-a7be8cc67bda.PNG) From here the flow of analysis is the same and the full in memory analysis wil be left as an execise for the reader. For special purpouse i will do in the analysis up to the point where almost the content process is done and right before the browser process start there i will stop. The purpose of this is just to showcase how ipc start communicating between each other. Now after the loading of the dll, we will have to look for a call rax instruction. what i did was to list 0x40 instruction from current eip after loading dll.![Captur123e](https://user-images.githubusercontent.com/25670930/146707045-696c1f81-663b-4d7a-b7ed-b46d810572a6.PNG). at that address actually is what we are looking for which is chrome!ChromeMain.![Capture123123](https://user-images.githubusercontent.com/25670930/146707128-f298d2fa-d90c-4384-b3f8-0092be7fe14f.PNG)
From there we want to break at ![zCapture](https://user-images.githubusercontent.com/25670930/146709631-7a95a673-58c3-466d-8b4a-6297decf61e6.PNG) which is basically a big check before jumping into content!content::ContentMain. from there we want to move over a few instructions and hit content!content::ContentMain. we want to step inside and break at content!content::RunContentProcess![Captuare](https://user-images.githubusercontent.com/25670930/146710883-9b474ede-1618-41bb-a9a5-a86c8fc44239.PNG) we want to step inside, and set a bp at content!content::ContentMainRunnerImpl::Run+0x430 and as soon as we step over ![Capture442](https://user-images.githubusercontent.com/25670930/146714548-934dc1e0-f994-40c0-90c0-ad99dfeb13e6.PNG) we see mojo ipc starting and we conclude that the next process aka browser process is responsable for actuall browser implementation and handling of all the other processes.![Captu1243re](https://user-images.githubusercontent.com/25670930/146714628-862e7ef3-b038-4658-bdb6-5b4db81dd378.PNG) 

===============================================================================================

2.


Last time we left of after understanding content process. today we will go for browser process
it's the process that is start after the content process and it's the second one in processes started by chrome.
Let's briefly describe it:
  *we can consider it the main process since the content process is more of a initialisation routine and some dependency checking,and it is started by the content process
  *it stays alive throughout the browser's entire lifetime.
  *It is the central coordinator of all the processes, and operates at the highest privilege level available to the browser.
  *since it runs at highest privillege levels in case other processes need to achive a higher level operation, that request is handled by browser process.
  *It controls features such as the address bar, bookmarks and the back/forward/reload buttons. Since this is the most privileged process it does not trust the data given to it by any of the other processes. 
  *also handles privileged operations such as UI, networking or filesystem storage for the other processes when necessary. 
          Now that we briefly know what it does it is time to go more indepth on it:
          Our journey starts at src\content\app in a file called content_main_runner_impl.cc at the function int ContentMainRunnerImpl::RunBrowser(MainFunctionParams main_params,bool start_minimal_browser) first operation it executes we see is a TRACE_EVENT_INSTANT0![Captureq](https://user-images.githubusercontent.com/25670930/147309302-2ee9a902-2201-4a8c-b68d-ec16e378eb23.PNG) . What is that ? Before that what is even a trace function? Well if we go to https://lwn.net/Articles/379903/ we can see they define it as ![Captur12348e](https://user-images.githubusercontent.com/25670930/147310480-5c3030d1-b5ba-4117-8347-c4e092376f99.PNG), doing a little bit of abstraction we can come to a conclusion that what a trace function is a function that records data at a specific point in a stackframe. It does more than recording the stack of execution function, it can record local variables of the last executed function.Now that we know what a tracefunction is let's see what our does. 
  If we go to https://chromium.googlesource.com/chromium/src/base/trace_event/common/+/refs/heads/main/trace_event_common.h we see that they define it as a function who's solve purpose is to "tracking application performance and resource usage". Going a little more indepth so we can understand what it does we see it's a macro and it is defined as ![Captureqq](https://user-images.githubusercontent.com/25670930/147309713-8394c04e-d28f-40a4-bb07-71d99201e361.PNG). We can see a short deffinition upper of the macro which says it record a single event called "name" immediately, with 0, 1 or 2 arguments. And in case the category of the belonging event is not enabled, it does nothing.We can see that as parameters we have "startup" as main category and as a subsystem we have "ContentMainRunnerImpl::RunBrowser(begin)" and the third paremets is TRACE_EVENT_SCOPE_THREAD which is "#define TRACE_EVENT_SCOPE_THREAD (static_cast<unsigned char>(2 << 2))" which i think based on the value id a id for a the respective instant event. so in essence what this does it simply traces(logs) the fact the we have entered with our  execution in the RunBrowser function . We than have a check to see if the browser main loop has already started and in case it has already started we exit the function ![Capturage](https://user-images.githubusercontent.com/25670930/147310843-4bb80348-be42-4bad-83cd-fc4246e49eda.PNG) . we than set a flag and after that we get to a rather interesting part of the code. We check if we have support for mojo_ipc mechanism and in case we do we use ShouldCreateFeatureList which creates a feature list with different processes tries to intialise them and finally tries to initialise mojo features.![Capture123123123](https://user-images.githubusercontent.com/25670930/147314140-568d2f6f-853f-456f-9ea8-6b2e7cdf235e.PNG). we than create a threadpool.![Captur``e](https://user-images.githubusercontent.com/25670930/147320243-1e20d2e3-ac62-4e0e-ae8e-d68a17cdeb2a.PNG) WTF is a threadpool???! Citying from wikipedia:"a software design pattern for achieving concurrency of execution in a computer program". More properly explained  "a thread pool maintains multiple threads waiting for tasks to be allocated for concurrent execution by the supervising program"(still this quote from wikipedia"). More properly expalined imagine a two lines of people working in a factory. Lets call them line a and line b. They are all supervised by a boss. let's call him line c. now line b has to wait for line a to finish their job and be notified by line c in order to work. and same goes for line a. And this is what a threadpool is. Here is also a small c example ![Captuqweqre](https://user-images.githubusercontent.com/25670930/147319801-98a4b383-d480-45b4-a0e7-4626f21f1d79.PNG). This is shamesly taken from https://stackoverflow.com/questions/15752659/thread-pooling-in-c11 .Next we have a call to PreBrowserMain(); which does some platform specific initialisation and after that we get to a point where we call BrowserTaskExecutor::Create(); ![Capture```](https://user-images.githubusercontent.com/25670930/147320325-2afe6abe-2f16-4fab-9051-21d3456f2c0b.PNG)
. Lets take a deep dive into what is does since the name is quite interesting and relying on an educated guess we can realise this might be interesting. Out detour begins inside of content/browser/scheduler/browser_task_executor.h where we find that BrowserTaskExecutor is a class which is  supposed to "to map base::TaskTraits to actual task queues for the browser process." Going a little down over the file we first  ![Capturea](https://user-images.githubusercontent.com/25670930/147321214-e390d172-b88d-40de-a95b-239270140d1b.PNG) it inherits from BaseBrowserTaskExecutor. So now in order to understand what  BrowserTaskExecutor we need to understand BaseBrowserTaskExecutor. we can see it inherits from TaskExecutor. Fortunatelly for us we see that it overwrites TaskExecutor's methods with overwriteable property which means that the'll be overwritten by other methods call later. But just for the currious people out there we can find it at base/task/task_executor.h and in case we inspect it we can find out  that TaskExecutor is a class which " can execute Tasks with a specific TaskTraits extension id ![Captureb](https://user-images.githubusercontent.com/25670930/147321539-66d5b748-b516-428c-8882-416c6e12573d.PNG)  . What is a task and what are tasktraits. We have already mentioned what is a task, but to refresh is one of a process of chrome's processes, and now what are TaskTraits?. They are located at base/task/task_traits.h and they are defined as follows "encapsulate information about a task that helps the thread pool make better scheduling decisions." Going back to our BrowserTaskExecutor. The method we call is Create() and it's analysis as it's follows ![browsertask_create](https://user-images.githubusercontent.com/25670930/147353158-2dca483e-fbcd-4237-baa3-3f3be48a8790.PNG) 
            first get check if the current task needs to run from a SingleThreadTaskRunner. aka if we need to run this task using an independent treadth.we do this by getting a pointer to tls. we than initialise an ui and & a thread scheduler. Basically here we init a scheduler for future events related to ui.![browser2](https://user-images.githubusercontent.com/25670930/147353764-ca3ce72e-ce01-4074-8113-cb1930630c75.png). And that's about it for that feature. moving on with the analysis of the content_main_runner_impl.cc we get here.![CreateVariationsIdsProvider](https://user-images.githubusercontent.com/25670930/147354423-580e14e3-31c5-4fba-8c85-2ef5ac52a5b2.PNG)
 searching for the variations ids provider class file we find it at components/variations/variations_ids_provider.h, there we see something reather interesting. it include a .mojom.h file. we can find it's definition at Debug/gen/components/variations/ variations.mojom.h ,which means it had to do something with the ipc mechanism. Now going over the actual definition of the class we see a comments which annotates as "A helper class for maintaining client experiments and metrics state transmitted in custom HTTP request headers." looking over it's behaviour and source definitionm se conclude it simply used as a marker for a function, which marks that "signed-in parameter supplied to GetClientDataHeaders()" which means somewhere in the stack execution there is a call to GetClientDataHeaders and later it will be called with a parameter. we than do     delegate_->PostEarlyInitialization(!!main_params.ui_task); which simply posts to mojo's ipc that it has to start the ui tasks. we do more initialisation ![b](https://user-images.githubusercontent.com/25670930/147355219-4964cc06-3389-4c72-a4de-a04bc6897783.PNG) and than we get to call RunBrowserProcessMain. ![a](https://user-images.githubusercontent.com/25670930/147355213-4f921b51-9b2a-48ee-b721-24b998893248.PNG) . If you hoped as me that this is where we see the gui starting , you are wrong . A quote from master oogwgay ![8ae6dca28db7d3afa6f483349c879962a437435d3e852e3fea35ef422acb95c1_3](https://user-images.githubusercontent.com/25670930/147355325-e5d8cbd4-d715-4d86-b64e-61c23059da4a.jpg). we than go and do some more checking and we hit BrowserMain![c](https://user-images.githubusercontent.com/25670930/147355407-23ca72a9-815a-48f9-9182-61e591859252.PNG). going over it it does some tracing and than we get to![init](https://user-images.githubusercontent.com/25670930/147356789-96a80fab-b51c-4d64-a4e4-a2e91020f4f3.PNG) which is where we actually start the gui. Now in case you don't trust me you will have to be a little more patient until we get to dynamic analysis. we than get to Run method which is the actual stuff we interested for that following that we will get to our next point for next point of understanding browser process. But for now let's take another short de-tour and investigate how Initialize method is created . Right of we see it traces the execution of init method and before that it creates a histogram![Capturez](https://user-images.githubusercontent.com/25670930/147362123-1dd3e636-3f9e-4790-b0dd-6bfe9f5ef35b.PNG). we than check the initialization_started_ flag to see if we got to initialisation  stage and in case we didn't , we init skia which is graphics library used by chrome,start a "timer" to count the second elapsed for running this method for later use in a histogram, check if we passed a param to binary to wait for debugger to attach to the process and finaly start a notification_service_ which by using an educated guess will notify the main watcher of threadpool when to start a service. ![Captureaaa](https://user-images.githubusercontent.com/25670930/147362430-6c6ef8bc-81a2-45a4-b037-e1682dca0c25.PNG). we than initialise the necessary fonts for chrome and create the mainbrowserloop, which is the coordonator for all processes and we jump over three methods which don't show any interest for us   main_loop_->CreateStartupTasks();
  int result_code = main_loop_->GetResultCode(); . from here what we are interested in to go into  CreateStartupTasks. from there we look at content\browser\browser_main_loop.cc, we are interested   startup_task_runner_->RunAllTasksNow(); which is inside createstartuptask method. starup_task_runner_ is a StartupTaskRunner which is located in same directory inside startup_task_runner.cc file. Now if we inspect RunAllTasksNow method we see what it does is simply iterate over all tasks and run them![run](https://user-images.githubusercontent.com/25670930/147379905-a2828e58-876b-4426-a439-e65b3e8f5ea8.PNG)

 ==========================================================================================
                                            Time for sone dynamic analysis
 
 Now in order to catch the renderer wil will have to start the binary in windbg. that is simply achivable by doing File->Open Executable and passsing --renderer-startup-dialog --no-sandbox --wait-for-debugger-children=renderer --renderer-process-limit=1 as arguments.![start](https://user-images.githubusercontent.com/25670930/147380385-5a9dcbf4-5601-4e2e-b880-b34dbd452093.PNG). we than set a bp at content!content::StartupTaskRunner::RunAllTasksNow+0x88 so we can catch the ipc messeges and later the renderer.just as a reference poin this is how your dbg should look after running it once ![content](https://user-images.githubusercontent.com/25670930/147380507-dbeda8be-2b93-4cdf-82c7-4294cf652025.PNG) . you should see the chrome is running in full browser mode message. from there you run counting from one to 5 so you run it another four times. and than you should see something like ![debuugz](https://user-images.githubusercontent.com/25670930/147380550-960aea41-2a8d-448f-97f9-a70eb1b9073e.PNG). i recommand using procmon so you can monitor the renderer process when it starts. after that you hook it into another debugger ,and in case you used the upper arguments you should see a message box popping telling you the renderer pid![helk](https://user-images.githubusercontent.com/25670930/147380575-1eeca73f-ea86-4f0e-8757-aa3c88a147c1.PNG). from ther you will want to bp at base!base::RunLoop::Run. Unfortuntally you can't break at content!content::RendererMain for some reason. maybe because we hook the process right after it exits the renderemain function and it's run by a thread. Regardless here is how the ipc looks after four runs.![ipc1](https://user-images.githubusercontent.com/25670930/147380616-1f107c77-45f5-4338-8e05-cd19f69de24f.PNG). This indicates that the gui is started. and here is how the ipc looks after renderer process starts.![renderer4](https://user-images.githubusercontent.com/25670930/147380635-680899c6-4555-4338-8302-59b2da39d14b.PNG). 

 
 ============================================================================================================
 
 3.Now for the second part of browser process analysis we got to the point where we will be able to understand and debug the renderer, but we won't go there yet. I explicitly left one more function to analyse after RunBrowserProcessMain, well theoreticall it's after RunBrowser, but as you may recal RunBrowser is a wrapper for RunBrowserProcessMain. Now what i left outside is that in the file called content_main_runner_impl.cc in the folder src/content/app there is another function which is called after we spawn the renderer and it's called RunOtherNamedProcessTypeMain. This process is incharge for running all the other processes.![Captureother](https://user-images.githubusercontent.com/25670930/147548808-5fcd1c18-87bf-4be7-aca6-b09d0de235eb.PNG). Unfortunatelly we will have to analyse it from inside the renderer. While it not might make sense now, it will when we do dynamic analysis. For now let's understand what happends in the code.We see it's prototype is ![Capturzaqe](https://user-images.githubusercontent.com/25670930/147589925-f9a48162-add1-49a1-a368-41e7e830e098.PNG), which indicates that it will take the arguments passed to cmdline, which process type to expect and a chrome delegate.We than get to the beginning of the function where we have a macro definition to check for platform specifics and check for which process to instantiate a event handler. aka to check to see if it runs some functions for console process or for the browser process. ![Capturqqqqqqqqe](https://user-images.githubusercontent.com/25670930/147590080-1bcaebed-e426-4478-8e6a-3ede7eb400cc.PNG). We than iterate over the passed processes and compare it to a list of knows processes and run the respective process. ![Captuqaxzcre](https://user-images.githubusercontent.com/25670930/147590609-0bbac184-5715-4793-80ec-4d80f02923af.PNG) In case we don't get the respective process than it's a custom process implemented by someone![Captureshaveica](https://user-images.githubusercontent.com/25670930/147590710-b05f7f7a-da54-49a5-915e-5feec1b2b210.PNG)











 
 
              
              
              
              
              


