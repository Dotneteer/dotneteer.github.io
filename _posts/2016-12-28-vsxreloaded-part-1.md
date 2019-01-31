---
layout: post
title:  "VSXReloaded — Part #1: How to Start Learning Visual Studio Extensibility"
date:   2016-12-28 13:20:00 +0100
categories:
    - Visual Studio Extensibility
tags:
    - VSX,
    - VSPackage
    - C#
---

It’s been a long time since I wrote my last Visual Studio Extensibility article, back in February 2009. That time—after about two years of active blogging about VSX—I turned to customers and projects that did not allow spending enough time with my favorite topic. Seven years of silence is too much. However, in the recent months, I have been working on projects where extending Visual Studio gives excellent value to the products my team is continuously developing.

I decided to reanimate my old blog and start writing posts about Visual Studio Extensibility again. This post is the first fruit of the new series.

### LearnVSXNow—Where the Whole Story Started

Nine years ago, I was a newbie in VSX. I eagerly wanted to learn the ways I can make Visual Studio better with my custom extensions, but I did not find enough information that would accelerate the learning process.

I was a fan of Visual Studio. Several times I flirted with the idea to create development utilities and integrate them with the IDE. I also wanted to convince my teammates to implement their tools as VS packages. However, when they asked me how to start, I always pointed to the VS SDK and its documentation. Most of them responded like this:

* “I don’t have time to go through hundreds of pages…”
* “I don’t know COM, and do not want to deal with that stuff; I want to use my .NET classes…”
* “I have already read the first fifty pages but no idea how to start quickly…”
* “I saw the SDK samples, but it seems challenging to create my package…”

I did not think that the only problem was the attitude of my fellows. They had already learned .NET, WinForms, ASP.NET, WCF, and other technologies, and they did it fast enough. I knew, they could to pick up VSX, too. What they told me was they had found that learning process painful and time-consuming.

As an MVP I decided to change the world a bit around VSX. I started a blogging project with the code name of “LearnVSXNow” and published about five dozen articles.

### Visual Studio Packages

Back in 2007, developers had a couple of options to choose from when developing custom Visual Studio Extensions. They had macros, VS add-ins, and Visual Studio Packages—aka VSPackages. Each of these methods used their peculiar approach to add something new to the IDE.

Since 2010, MEF extensions have been available to create custom tools that integrate with the code editor.

As of today, only two ways remained: VSPackages and MEF extensions. In this series, I will treat the MEF option only tangentially, and focus on VSPackages.

There is no doubt developing VSPackages is the most powerful way to add functionality to the IDE. The clear evidence for this is the fact that the entire Visual Studio integrates VSPackages into the shell. All the languages, the code editor and the designers, the debugger, the project system and many other components are packages. Figure 1 shows the About Microsoft Visual Studio dialog that lists the packages integrated with the IDE. Most of them come with Visual Studio out-of-the-box, while others are installed separately.

![VsAbout](/../assets/images/VSAbout.png)

*__Figure 1__: The About dialog of Visual Studio*

From developers point of view, adding a new package to VS is just like extending the core functionality of the IDE with a component, as if Microsoft developed that. The IDE does not make any distinction between Microsoft-created and third-party components; users can use each of them as a part of the VS IDE.

Packages are binaries developed with your preferred language (C#, VB, F# or C++), so from intellectual property guarding aspect they can be as safe as other .NET binaries.

Packages can be encapsulated into a VSIX file, which is the unit of deployment for a VS extension. Visual Studio recognizes this format and installs the contents of the file to the right location.

Extensibility Points
Practically, there are no limits on the functionality to extend Visual Studio. You can create tools that deal with the code just as a visual designer for a particular kind of file. If you’d like to see a new programming language of yours in the IDE, you can create and integrate it with the shell.

Visual Studio has dozens of extensibility points where you can add something valuable. Just for whetting up your appetite, let me list a few of them:

You can add new menu items and commands to the IDE. When you activate such a menu item, the command executes.
You can extend tool windows installed with Visual Studio or create your patch of UI encapsulated into a new tool window. For example, you can add your own pane to the Output windows, create a Toolbox control or extend the status bar.
With the extensible project system, you can create your project template or project type.
You can add settings that integrate with the Tools | Options dialog, and even save them in your projects.
The extensible editor offers you several ways to customize the code editor. The .NET Compiler Platform (aka Roslyn) offers you project types to create code analyzers and other tools that let you parse and modify the code.
Where to Start?
The best way to getting started with Visual Studio Extensibility is to understand what packages are and how they work together with the IDE. In the next posts, I will dissect this topic.