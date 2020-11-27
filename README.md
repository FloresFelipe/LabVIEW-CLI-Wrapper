# LabVIEW-CLI-Wrapper

___Easily create your own API by wrapping other applications Command Line Interfaces (CLI).___

## Introduction

You should probably be familiar with the __System Exec VI__ which is used to [run commands in the OS console](https://knowledge.ni.com/KnowledgeArticleDetails?id=kA03q000000YGivCAG&l=en-US).

![System Exec.vi view from LabVIEW Context Help](/Documentation/images/sysexec.gif "System Exec VI")

This single VI gives you the power to easily run any command you would run in the windows command prompt, for example, directly from the LabVIEW block diagram. Some common tasks you can perform with it are:

* Run External Executables.
* Execute OS commands such as creating directories, listing files, retrieving IP configuration, schedule the computer to shutdown, etc.
* Run Commands from applications' command line interfaces.

Lately, I've been working with the [InisghtCM Console](https://www.ni.com/en-us/support/documentation/supplemental/17/using-the-insightcm-console-application.html) application, which is a command line application for managing the [InsightCM server](https://www.ni.com/en-us/shop/data-acquisition-and-control/application-software-for-data-acquisition-and-control-category/what-is-insightcm.html). Using it is not rocket science at all, but I wanted to incorporate a GUI to it in order to have a more friendly user interface and will keep me from repeating the same instructions several times in the command line. :v::smiley:

Well, I started by building an Event-Driven State Machine that basically responds to a button click event by building a command string, running it by calling the System Exec.vi, and then processing the standard output to show it on different types of indicators (List Box, String Indicator, Menu Ring, etc)

It worked fine, but I began to think that this code could be easily reused to bring other applications that operate via CLI to LabVIEW. But how to do it in a scalable manner? I mean, do I need to copy the entire project, delete the cases for the InsightCM command, edit both the TypeDefs in the project (Data Cluster and Enum) to then be able to implement my commands and UI? That sounds like a lot of work! :weary:

When I think in reusability and scalability, LabVIEW OOP always comes up to my mind, so I decided to give it a shot.

## From Reusable State Machine to LabVIEW Classes.

### Where it all began

For sometime I was afraid of giving away of my handy state machine design pattern, as it is quite easy to implement and solves most part of my development challenges. Moreover, talking about LabVIEW OOP sounded like too much for my needs. In the last months, I've been giving away of those thoughts and developing everything I can in LabVIEW OOP.

In my state machine, I noticed there were three fixed states that will always be there and they will always be executed in the same order:

1. __Build Command__ - Never changes. It always formats a command in a ``[app name] [command] [switches/options]`` structure, and concatenates an end of line to it.
2. __Run Command__ - Never changes. It calls the __System Exec.vi__, passing the command built and the path from where we must run it.
3. __Process Standard output__ - May change depending on the application output, but will always be called after the Run Command State to process the string return coming out of the System Exec VI.

That order of steps, where some of them will be always the same and some of them may change depending on the implementation reminds me of the __Template Method__ design pattern. This is one of those [Object Oriented design patterns by the Gang of Four (GoF)](https://en.wikipedia.org/wiki/Design_Patterns). This design pattern basically states that a class should implement a method that invoke other methods in the class in a specific, standardized way, that is, this _template method_ will tell the users of the class how must the steps run.

Another challenge is that some initial parameters such as the __Application Name__ and the __Working Directory__, as well as checking if the app is installed, its bitness, and any initial validation may vary depending on the application, as well as could be too much for a user that wanted an easier alternative to the CLI. Thinking on that, I decided to implement another object oriented design pattern called __Factory Method__. That design pattern determines that you should create method in a class that will return a parametrized, "ready-to-use" object, hiding all the complexity and making sure all initialization routines were done correctly.

I highly recommend you to review the [Template Method Pattern](https://refactoring.guru/design-patterns/template-method) and [Factory Method Patter](https://refactoring.guru/design-patterns/factory-method) articles at __refactoring.guru__. The text is easy to understand and there is a lot of good illustrations over there.

### Designing the Base Class

As those design patters are quite simple to understand and implement I started off by designing the __Console Wapper__ class (I know. I need to get better at naming things in my projects :cold_sweat:)

![Console Wrapper Class](/Documentation/class%20diagrams/Console%20Wrapper.png "Console Wrapper Class")

where __Initialize__ is my factory method, __Execute__ is my template method and __Build Command__, __Run Command__, and __Process Standard Output__ (if you remember well a few lines above) are the methods I wanted to reuse.

The attributes of the class were taken from the Data Cluster TypeDef from my State Machine.

__Note:__ I did not put the member accessors in the class diagram just to make it clearer, I implemented them in the actual class! :smiley::thumbsup:

With that in mind I created a LabVIEW class called __Console Wrapper.lvclass__

![Console Class In LabVIEW](/Documentation/images/ConsoleWrapperClassInProject.png "class created in the LabVIEW Project")

Take a look at the __Execute__ method block diagram (__Here is where reusability comes into play__ :exclamation::grey_exclamation:):

![Execute Method Block Diagram](/Documentation/images/ExecuteBD.png "Template Method implemented")

Both __Build Command.vi__ and __Run Command.vi__ are static dispatch (i.e., all classes will run their unique implementation). In turn, __Process Standard Output__ provides a default implementation, which is showing the string output in a _one button dialog_, but allows descendant classes to override it.

If a descendant class wants to execute a command, all it needs to do is parametrize the command and call the __Execute.vi__ and let it taking care of the rest!

The __Initialize.vi__ is also Dynamic Dispatch and it needs to be overridden by descendant classes. I implemented a default behavior of entering the ``dir`` command in the C:\ directory.

### Running the First Test

As simple as it can be, I created a unit test to validate my template pattern, using the factory method as my "Setup VI". To do so, I used the [LabVIEW Unit Test Framework](https://www.ni.com/en-us/innovations/white-papers/09/prove-it-works--using-the-unit-test-framework-for-software-testi.html) as it helps me to test my modules automatically, without needing to create more VIs for that.

What it does is similar to the following code, but with more functionality such as test statistics and again, __no need of creating test VIs__

![Base Class Test Code](/Documentation/images/initialTest.png)

Here is how the unit test runs.

![Execute Method Unit Test](/Documentation/images/FirstUnitTestExec.gif)


## Extending the Console Wrapper class

### Overrides

After validating the base class, I wanted to extend it to create Application Specific Console Wrappers. The first class created from the Console Wrapper class was the __InsightCM Console Wrapper__ class.

![InsightCM Console Wrapper Class Diagram](/Documentation/class%20diagrams/InsightCMConsoleWrapper.png "InsightCM Console Wrapper Class")

First, I created that class in LabVIEW and overrode both __Initialize.vi__ and __Process Standard Output.VIs__ with specific implementations for this App, and then I started developing the methods for the specific CLI commands. I created one method for each command I needed to implement.

__Note:__ Some commands may run with different set of options. In that case, you can either use the same VI with some decision logic or you can turn that method into a Polymorphic VI.

The __Initialize.vi__ sets the __Application Name__ and __Working Directory__ attributes.

![Initialize Override](/Documentation/images/initialize01.png "Initialize Override")

The __Process Standard Output.vi__ override will parse the command response string and get the list of definitions.

![Process Standard Output Override](/Documentation/images/ProcessStandardOutputOverride.png "Proces Standard Output Override")

__Note:__ The __Initialize__ method is a public method because it must be called in any code that will use that your wrapper class. On the other hand. the __Process Standard Output__ method is protected since we want it to be called only from the template method at the Base class while allowing it to be overridden by descendant classes.

### Implementing Application Commands

The InsightCM Console application has a command that exports data sets from the server as TDMS, taking as inputs the asset you want to get data from and the start and end time when that data was collected. It is called __exporttdms__.

| Command Structure | Example |
| :---- |:----|
|``exporttdms -dir [directory] -asset [top-most equipment in path to sensor] -start [start date "yyyy-mm-dd hh:mm:ss.S"] -end [end date "yyyy-mm-dd hh:mm:ss.S"]``|``InsightCMConsole exporttdms -dir C:\temp -asset "Plant A\|Turbine Generator" -start "2019-07-24 22:00:00.0" -end "2019-07-25 01:00:00.0"``|

Below is how I implemented it in LabVIEW:

![TDMS Export Data Events.vi](/Documentation/images/TDMSExport.png "TDMS Export Data Events.vi")

Other command implemented was the __listdefinitions__ which returns a list of type definitions available in the server.

| Command Structure | Example |
| :---- |:----|
|``listdefinitions -t [type]``|``listdefinitions	-t device``|

LabVIEW implementation:

![ListDefinitions](/Documentation/images/ListDefinitions.png "ListDefinitions.vi")

Unit Tests were also created for each of the "command" methods in the class. The final version of the class looks like this.


![Class in the Project](/Documentation/images/ICMConsoleWrapperClassInProject.png "InsightCM Console Wrapper Implementation")

## Comparing CLI Vs LabVIEW Code

See how ``listdefinitions`` is run from the command line.

![listdefinitions from cmd](/Documentation/images/listdefinitionsFromConsole.png "listdefinitions from the command line")

Now how it goes with LabVIEW.

![listdefinitions in LabVIEW](/Documentation/images/ListDefinitionsLV.gif "listdefinitions with LabVIEW")


## Other Overrides

I ended up implementing other sub-classes with the goal of validating the Base Class extension. I created a Wrapper for the [7-Zip File Compressor](https://www.7-zip.org/) and for the [VLC Media Player](https://www.videolan.org/), both with minimal functionality. Feel free to pull request and add more features to it.

![Other implementations](/Documentation/class%20diagrams/AllClasses.png)

I encourage you to clone this repository and take a look at all implementations.


### Next Steps and Improvements

- [ ] Instructions and Examples on how to and create your and deploy your own API.
- [ ] Improve the Error Handling to display a more precise messages (probably including the error return from the commands).
- [ ] Improve the Factory methods of both 7-zip and VLC to support both 32 and 64-bit installations
- [ ] Add other commands to the InsightCM Console Wrapper (I want to create a Polymorphic VI for the exporttdms command so it supports both time and event id inputs)

---

:email: __Feel free to send me comments and improvement suggestions on [flores.felipegomes@gmail.com](mailto:flores.felipegomes@gmail.com)__

---
