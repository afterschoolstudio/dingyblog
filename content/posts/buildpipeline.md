---
title:  Build Pipeline
date: 2021-03-03
tags: ["Kinc", "Karp", "Kha", "Dotnet"]
slug: build-pipeline-structure
---

I’ve recently strung out the MVP of Dinghy. I’m compiling Kinc with a set of extensions I wrote in C into a dll, then calling those functions with C# bindings I wrote. The C# bindings ("Karp") call that compiled library code. Karp is hand-written for now, but I’m hoping at some point to have the bindings generated as part of the same build process.

Bringing those two DLLs (Kinc + Karp) over into a new Dotnet project and running the test function in Karp, produces this!

{{< figure src="/images/firsttri.png" caption="First triangle in Dinghy" >}}

Early days for sure, but as any engine programmer knows the triangle is a big step!

Before focusing on more engine related work, right now I’m really considering the build pipeline itself, as it acts as the framework upon which everything else relies upon.

It’s something that I think is easier to think about in terms of desired end effect, which is that a user can basically bring in a single DLL (Dinghy.dll) into a new dotnet project and that should work everywhere and contain all platform specific dependencies.

Getting there though starts with working from the other side of the pipeline and trying to reverse (forward?) ourselves into that desired state.

## Kinc
{{< figure src="/images/kincdll-pipe.png" caption="Producing the Kinc DLL" >}}

[Kinc](https://github.com/Kode/Kinc/tree/63b9589eb0e6da0c20a833c7a5f2a258b6fc7e3d) is the backend of Dinghy, and gives us a unifed API to access different platform specific APIs. It's used, maybe most prominently, on [Kha](http://kha.tech), which is then used on a lot of different things, maybe most prominently [Armory](https://armory3d.org).

When you build a project with Kinc, you build for a target platform and it "just works". It’s the core vale prop so I don’t expect Kinc to be a “compile once, run anywhere” sort of thing (it doesn't use a VM). However, this means that I need to compile my extended version of Kinc for every platform that Dinghy will support. This isn’t hard, as Kinc projects are built with a handy tool called Kincmake, which will generate the requisite platform project that can be compiled into a library. Kincmake also does one other special thing that I’ll talk about later.

For now though let’s assume I’ve compiled my flavor of Kinc and now have a folder full of platform-specific libraries. This looks a lot like [this repo](https://github.com/Kode-Community/Kinc_bin), but again *my* Kinc is a bit special so I can’t use those directly (they just compile raw Kinc).

## Karp
{{< figure src="/images/karpdll-pipe.png" caption="Producing the Karp DLL" >}}

Karp (Kinc in CSharp) is the set of bindings I wrote/am working on that binds C# calls down to the low level calls in Kinc. If you were using Karp directly (and I fully expect/hope that people will!) you *could* just include the Kinc library file for the platform you’re developing on.

I actually don’t know and need to test this, but I’m assuming here that I can’t use a Kinc-produced dll on a Mac and instead need to use a dylib? Or maybe because I’m in Dotnet land all I actually need is the DLL? I’ll have to test. I'm still learning! Because I'm using Dotnet, if I was able to just use a DLL all the way up that would be great.

For now, I give Karp all the compiled libraries (just place them in the directory). The bindings I wrote call down into the platform library and give us that beautiful triangle above.

## Dinghy
{{< figure src="/images/dinghydll-pipe.png" caption="Producing the Dinghy DLL" >}}

Dinghy, at a baseline, just uses Karp! It’s the first Karp-based app, but like I said above I fully expect people to build other applications that sit at this same part of the stack. I'm recalling similar stacks in the Haxe ecosystem like OpenFL and Lime (and Kha) that are all usable directly but are more often wrapped in more accessible (or specific) frameworks:

| "Low Level" Framework | "High Level" library that wraps that framework |
| --------------------- | ---------------------------------------------- |
| Lime                  | OpenFL                                         |
| OpenFL                | Flixel                                         |
| Kha                   | Armory                                         |
| XNA                   | Monogame                                       |
| Monogame              | Nez2D                                          |

And so on. Any of those are usable directly, but people, depending on their appetite for control vs. accessibility, have to pick somewhere in the stack to start making stuff.

Dinghy, I expect, will fall somewhere between Kha and Nez2D (which is admitedly a wide gap). I’ve mentioned Heaps before, and if I’m being honest I’d be happy if Dinghy just becomes a C# version of Heaps. But I also look to the alpha of Luxe as a great target. It shouldn't feel *too* low-level, and should be geared for productivity, but also give you access all the way to Kinc if you want it.

Also in case it’s not apparent, **Dinghy doesn’t use XNA/FNA/Monogame**. The goal here is provide as low level a wrapper to a cross-platform backend that allows you to use the latest versions of C# as soon as they are available. I don’t want anything about Dinghy to be “legacy” and instead I want it to be forward facing. Dinghy should feel like a *new* way to make games in C# instead of one that inherits past constraints or design paradigms. There’s also no reason someone couldn’t just reimplement Monogame’s API on top of Karp (maybe you reading this should!)

Dinghy will ingest those Karp libraries and then use Karp bindings to call lower level graphics functions, and also provide an actual API to applications that use Dinghy. These will all be appropriately namespaced so using them in an application should be pretty easy.

Dinghy bundles all the platform libraries and compiles *itself* down to a dll, so, ideally, the engine is a single library. At this same level of abstraction we could bring in other libraries to extend Dinghy’s core features and functionality ("Goodies", like a physics library), but ideally the engine itself isn’t too bulky and is portable.

## Your game/application
{{< figure src="/images/gameexe-pipe.png" caption="Producing your game" >}}

So finally we’re back to where we wanted to go, your application! Ideally here, you can simply include the Dinghy Dll in a new Dotnet project, and get started making... something!

Dinghy is nominally a game engine/framework, but I’m really interested in running some performance tests on this stack to see if it could be useful for a more generic “media application framework”. I know Kinc/Kha are fast and Kha bills itself as a media framework, so I’m interested in seeing how much overhead Dotnet introduces.

Once you’ve got your program written though,you’ll use dotnet’s publishing features to build platform specific executables that allow people to run your application! I have some thoughts here as well about simplifying this process (recalling Heaps’ .hxml format), but ideally this should be a single build command.

### Shaders/Assets

Remember before when I mentioned that, all the way back at the start of the stack, Kincmake does something else? When you’re building a Kinc project, it sort of assumes that what you’re building is the final artifact, ie, that after that build step you’ve got your completed application *for your target platform*. We just walked through our general build pipeline, and can see that that’s definitely not the case — we don't know our final platform until someone builds their Dinghy-based app. However, when Kincmake runs in that first step, it also *cross-compiles shaders* to the target platform. This is big.

Kinc, via a tool called [Krafix](https://github.com/Kode/krafix), allows you to write shaders in GLSL and have them cross-compiled to the graphics framework of your choosing. However, at this point in the Dinghy build process, we have no idea what platform an end user is going to be using, and obviously have no shaders!

I don’t want to lose this part of the pipeline, so we need a way to basically surface it as part of the final *dotnet build* command an end user runs. I have some ideas on how to do this (namely, that Krafix itself can be compiled into a library), but I’ll muse on that in another post more specifically about assets, as this post is already much longer than I anticipated.

## Overview
So considering all the above, here’s the general stack of Dinghy:

{{< figure src="/images/dinghy-pipe.png" caption="Dinghy Stack" >}}

I’m really liking where it’s landing and it feels like the right combination of robustness and conciseness. I’ll have more in a week or two about whatever I’m working on next, but for now I hope this gives you a deeper look at what I’m aiming to do with Dinghy.