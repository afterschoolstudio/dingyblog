---
title:  2023
date: 2023-10-02
tags: ["Sokol", "Arch", "Alpha"]
slug: 2023-update
---

Well hello again. Been a minute eh?

Turns out making [a whole videogame](https://store.steampowered.com/app/690370/Cantata/) is a lot of work, so the amount of time I've been able to dedicate to *any* side projects had been somewhat minimal. Doing something as "big" as Dinghy is even harder to find time for, because there's so much initial exploration and so little return.

There HAS been work though, especially since Cantata's release, so here's something I'm happy to show:

{{< figure src="/images/ding2.gif" caption="Dinghy Demo" >}}

I'll talk about this more in a little bit, but first wanted to talk a little bit about *thinking* instead of just coding.

Something that working on Cantata forced me to do, just given time contention, was that the time I could spend on Dinghy was never enough to really start _coding_ it. So instead I spent a lot of time *thinking* about it, channeling [Rich Hickey's idea of "Hammock Driven Development"](https://www.youtube.com/watch?v=f84n5oFoZBc). I have a lot of project ideas, but not a lot of time, so my strategy for focusing on something/choosing something to actually pursue is to, almost like spaced repetition, return to ideas over time to see if they "stick". If they don't stick, I'll stop letting it take up mind space, and if so I'll think a little more about it. I write down nearly every idea I have with this in mind, knowing that 90% of them will fade beyond that initial spark, but some will indeed rise to the top. I continuously circle around the ideas and try to re-surface them to myself often enough that I can reevaluate if I still like the idea, and if so, think more about it.

Keeping this running catalog of ideas is especially important during the throes of another project. Big projects take up all of your time, but then can all at once it stop, and then that vacuum of newfound free time can easily absorb even the most ill-advised idea that may have just happened to pop into your head around the same time. What you want is that when you free up, you start working on the *right* next choice. And after many rounds of thinking over it and it continuing to stick, *Dinghy* is definitely one of those things.

Dinghy consistently passes that test of "stickiness", and over time feels like what it's trying to do is more than just convenient but instead, [vital](https://www.theverge.com/2023/9/22/23882768/unity-new-pricing-model-update). An exodus of C# programmers from Unity, especially people that do 2D, are looking around and trying to find what makes sense to go to next, and there honestly aren't enough good answers. Dinghy is trying to be one of them. It's meant to be a simple, cross platform framework for C# that you can rapidly prototype games and interactive things in, but also have bones that can scale to larger projects. Seems simple enough, and all the pieces are really laying there, but nobody else has done it!

I think also really just sticking to 2D and never trying to touch 3D is really helping me maintain that focus. If you assume 2D, you can do a lot of stuff around engine/API UX that wouldn't make sense for a 3D engine, but also would make there be a lot of "API noise" for people doing 2D things in an engine that supports 3D. So we're keeping it trim and to the point.

So *what* has actually happened? Well, a lot!

## Sokol

Probably the biggest change from the previous blog posts and is that I moved from Kinc to [Sokol](https://github.com/floooh/sokol). There's a few reasons for this:

* Sokol is generally further along than Kinc and feels less like a prototype.
* The primary maintainer of the Sokol repo is really friendly and responsive
* More people in general know about Sokol and use it. For someone like me that knows so little, being able to ask about Sokol to other people besides the repo owner is really nice.
* Sokol has *tons* of examples, making learning the headers a lot easier! The documentation is also embedded in the source itself, which is weird at first but then turns out to be really nice — you just search what you are looking for in the IDE and can instantly pull up the docs about it.
* Sokol has lots of built in support for other libs (DearImGUI, etc.) that are mostly drop-in.

Related to that last point, Sokol has a lot of nice little things that really help out engine dev. The DearImGUI bindings are obviously big, but then other nice examples like the [Fontstash](https://floooh.github.io/sokol-html5/fontstash-sapp.html) interop, [Debug Text](https://floooh.github.io/sokol-html5/debugtext-sapp.html), and upcoming WebGPU bindings, make me feel good about "living" in the Sokol ecosystem.

**Sokol_GP**

As a small aside, I'm also primarily using [Sokol_GP](https://github.com/edubart/sokol_gp) for the actual 2D drawing — it's basically a simple modern 2D drawing lib that sits on top of Sokol mostly with ease of use. You can mix and match Sokol's core libs with GP, so it's been pretty nice!

## ClangSharpPInvokeGenerator

I've also moved to using the [ClangSharpPInvokeGenerator](https://github.com/dotnet/ClangSharp) to generate the PInvoke code to bind C# to Sokol and other libs. In the past I was mostly handwriting all the PInvoke code.This was... insightful, but incredibly tedious and really prone to error. It was nice to get my head around the whole Marshalling/PInvoke workflow, but one thing you realize in doing that is that there *is* a right answer to any binding function, and as such generating them is far more efficient. I also learned more about the concept of "blittable" bindings (aka, no runtime Marshaling), which pushed me to adopt the new generator because it directly spits out those bindings!

Working with blittable bindings is great for performance, but it does mean the API surface of using those functions is a bit more involved. Starting up Sokol for example looks like this:

```c#
unsafe
{
    var window_title = System.Text.Encoding.UTF8.GetBytes(opts.appName);
    fixed (byte* ptr = window_title)
    {
        //init
        sapp_desc desc = default;
        desc.width = opts.width;
        desc.height = opts.height;
        desc.icon.sokol_default = 1;
        desc.window_title = (sbyte*)ptr;
        desc.init_cb = &Initialize;
        desc.event_cb = &Event;
        desc.frame_cb = &Frame;
        desc.cleanup_cb = &Cleanup;
        desc.logger.func = &Sokol_Logger;
        App.run(&desc);
    }
}
```

Pointers? Fixed?? Address references??? Doing it this way does mean you've got to dive headfirst into `unsafe` waters, which (for me at least) meant a large onboarding period of just wrapping my head around this "type" of C# programming. But now that I'm on the other side of that cost, [I'm much more comfortable with it](https://blog.kylekukshtel.com/csharp-lowlevel-memory-pinvoke-span-blittable-bindings)! C# can also nicely encapsulate this work, so for anyone who uses Dinghy they don't need to worry about any of it. Working in Dinghy is more "normal" C#.

As a side note, I think I may write a small post on [my main blog](https://blog.kylekukshtel.com) about using the generator. The docs are a bit hard to parse so I think it could be useful to see a common use case in action.

## Zig Build

So I've got nice new bindings for Sokol, but how about actually compiling the headers?

Previously, the build system was a bit weird. But weird in a way that I imagine any cross-platform library is. Building the headers meant generating platform-specific project files for the dependent platform (Visual Studio/MSVC on Windows, XCode on OSX), then basically building those projects from the command line version of those tools. This seemed sort of inescapable, and I guess is nice for some use cases, but I always found it sort of jank.

But then I learned about Zig, and more specifically that Zig ships with a drop-in, cross-platform C compiler replacement. I tried it... and it works?? It feels like magic. So now all I basically do is just run zig build (with the correct Zig script) and out comes platform-specific DLLs.

I'm also lucky here because Sokol has actively maintained and auto-generated Zig bindings, and also happens to have a [Zig Build script](https://github.com/floooh/sokol-zig/blob/master/build.zig) that I can copy and modify to build a DLL instead of an exe. Thanks Andres!

Worth saying as well that I am generally digging Zig. Dinghy is still very much focused to be a C# library, but I can definitely see myself [dipping into Zig + Sokol land on the side](https://github.com/floooh/pacman.zig/blob/main/src/pacman.zig).

**General Lib Workflow**

This pipeline also really works for other [STB](https://github.com/nothings/stb) style libs, so I've started to incorporate some STB headers in as well. stb_image specifically is now in the engine for image loading, and I'm eyeing others in there as things grow out. I feel like with STB and other [single-file header](https://github.com/nothings/single_file_libs) libs (like [the Cute ones](https://github.com/RandyGaul/cute_headers)), I'll have most core functionality I'll need at the C level and all the wrangling of libs themselves at the C# level.

## ECS

Everyone's favorite game development buzzword. I'm making Dinghy be an ECS-oriented engine (via [Arch](https://github.com/genaray/Arch)). If I'm honest I'm just trying this on a bit. I'm not fully committed to it yet but so far it's a nice paradigm and it *is* fast. I think people discount how many objects are often in 2D games compared to 3D games, so it does feel like gaining performance from this paradigm makes sense.

You *can* circumvent the ECS (or everything?) in the engine if you want, but I intend to make all the learning material/tutorials/etc. try to repeat and show the same paradigms. The "critical path" of the engine should feel well-trodden and *smooth*.

One thing I do like about Arch is that, in addition to just the general quality of the library itself, it has some Dinghy-aligned features (specifically "magic") that I think will be nice to have via it's Extensions package and the source-generated Queries (aka systems):

```c#
[Query]
[All<Player, Mob>, Any<Idle, Moving>, None<Alive>]  // Attributes also accept non generics 
public void ResetVelocity(ref Velocity vel)
{
    vel = new Velocity{ X = 0, Y = 0 };
}
```

The actual query code is source-generated against that function + attributes, and reduces the overall need for boilerplate.

## Actual Engine Screenshot

So let's talk about what we're seeing here (It runs smooth I promise, just GIF recording, etc. etc.):

{{< figure src="/images/ding2.gif" caption="Dinghy Demo" >}}

Hell yeah! So at a baseline, the key thing is that, **all you need to get what you see on the left is *just* the code on the right** (provided you have the Engine DLLS obv, etc. etc.) No platform setup code, no `GraphicsDevice` to worry about — You just write the code on the right, and you can do what's on the left. Ideally we can make it even more terse, but for now you get the idea. The code is basically *all signal* with minimal (no?) boilerplate.

What's especially cool about this is that, a few years ago, I wrote fake Dinghy code to show how I wanted game authoring to be, and this basically meets that target. It's also not like it's sitting on a ton of abstractions just to prove a point — this is all workhorse code that you would probably write the same if it's for your project.

### Low "Ceremony"

Jon Skeet (blessed be he), has this concept of "ceremony" in programming. The idea of it is sort of ancillary noise that has to exist next to code that actually *does something*, such that the "ceremonial" code isn't doing much besides scaffolding the core idea. So the idea is that ideally you can reduce how much ceremony which is required at a baseline, but, when you want to roll out the red carpet, you *can* still bring it back.It's the difference between old C# style "high-ceremony" main function entry:

```c#
public class Program {
	public static void Main(string[] args) {
		Console.WriteLine("Hello World")
  }
}
```

And the "low ceremony" new style top-level statement entry:

```c#
Console.WriteLine("Hello World"); //literally all you need in a Program.cs file
```

You *can* still do it the old way, but all the code is basically boilerplate and is the same for 99% of use cases, so instead of requiring ceremony, you can just *do* the thing you intend. Default instead to a common base case, with opportunity to expand into full ceremony if necessary.

**Dinghy.Quick**

Dinghy is very much crafted in this mold, and to that end is the `Dinghy.Quick` namespace, which this the code above uses. It's "you'll most likely do it *this* way" style functions and wrappers that are meant for quickly scaffolding games, and gets at that Dinghy tenet of "Immediacy" (with a dash of "cozy"). It takes barely two lines of code to get a sprite on screen, and nothing about the function constructors or method calls are ambiguous about what they output is.

You *can* use all the Engine APIs yourself (and you will for any large project), but the idea is to onboard new people really easily with the most likely versions of functions and data that they will use, and you can change it up when you outgrow them.

So instead of writing:

```c#
InputSystem.Events.Key.Down += (key) =>...
```

You can write:

```c#
OnKeyDown += (key) =>...
```

### Workflow

Similarly, one of the key things I've been working on here was to really understand the development loop for Dinghy, and asking myself what does it mean to work *with* it.

I think this is related to a thread drawn from Dinghy's idea of "magic", but the more I think about games the more I really just end up thinking about "how can we make working with Data better?" 

> Show me your code and conceal your data structures, and I shall continue to be mystified. Show me your data structures, and I won’t usually need your code. – Eric Raymond

Game programming (and generally working with game engines) is so wrapped up in thinking about *code* that I want to try to frame (to users) that working with Dinghy is less about *writing code* and more about just *hooking up data*. Yes, there will be still be scripts and whatever, but thinking about, and getting users to think about, data from the *very* start of the project will I think be something really useful.

It's related to the idea that good data and dumb code is almost always better than bad data and good code. Dinghy wants to try and always give you Good Data, so that hooking it up for games is trivial. Going back to the idea of ceremony, it means looking at data as something that can be *directly actionable* instead of needing to wrap it in boilerplate.

There's the murmurings of an asset pipeline in the engine now (but also where a large focus of the "magic" I want to be is at), but the gist in terms of the current implementation is that external assets are wrapped up into data containers, and those data containers then have the ability to instantiate the most likely version of themselves into the world.

Dinghy will provide a lot of the common 2D ones out of the box (Textures, Sprites, Tilemaps, Animations, etc.), but you also easily author your own and tie those into the "import" pipeline. You can also easily compose data structures together (they're just classes) so if you need larger composite structures you can build those out as well.

> As an aside, I just wanted to mention how this is different from things like Unity out of the box. In Unity, you import an asset. That asset is *just* an asset and can be theoretically used for anything. In Dinghy, we assume that all data is instead *actionable* via it's own definition of an asset. Once an asset is in Unity, it's just loose sitting around, and then is attached to a component. That component can be on literally *any* type of object, that can then also contain literally *any* other components. On one hand, it's very flexible, but on the other hand it's very prone to [foot-gunning](https://en.wiktionary.org/wiki/footgun) and misconfiguration, as there are basically no guardrails towards doing what is *likely intended*. Dinghy instead is meant to be more rigid, and provide you with *guardrails geared towards productivity*, with the ability to move outside them if you want to.

**Data -> Entity**

So for a simple example, we've got SpriteData, which takes in a reference texture name and some data about where the sprite should go on screen, great. Here's SpriteData currently:

```c#
public record SpriteData(string texture,int startX = 0, int startY = 0) 
  : EntityData {
    public override void GetEntity(out Entity e)
    {
        e = Engine.World.Create(
            new Position(startX, startY), 
            new SpriteRenderer(texture));
    }
}
```

Notice that Data is a `record`, meaning that the class is meant to be immutable. It's just a container for information. However, by being data you have to define some notion of what it means to consume you. If you're a SpriteData, that means you're probably making a Sprite, and as such there is a direct function implemented to do just that.

You can still have non-object data, but the goal is to have no "loose" stuff lying around — everything should be data build towards entities, *or* data that gets directly incorporated into the development experience via [magic] code, but more on that in a future post.

**Entity -> Components**

Once you've got an entity, you add some components to it to give it behavior. SpriteData's Entity gives you a Position and SpriteRenderer by default, but you can also add components after the fact ([Arch has a few ways to do this](https://github.com/genaray/Arch/wiki/Performance-Features#bulk-adding)):

```c#
SpriteData logo_img = new("logo.png");
var logo = Add(logo_img); //adding the logo to the scene
logo.Add(new Velocity()); //adding a velocity component
```

In addition to components, you *can* also do non-component behavior modification to the entity as well, so you *can* use the ECS to-taste:

```c#
OnKeyDown += (key) =>  {
	ref var vel = ref logo.Get<Velocity>(); //arch's way to grab component data
	(int dx, int dy) v = key switch {
		Key.LEFT => (-1, 0),
		Key.RIGHT => (1, 0),
		Key.UP => (0, -1),
		Key.DOWN => (0, 1),
		_ => (0, 0)
	};
	vel.x += v.dx;
	vel.y += v.dy;
};
```

If I was building this out as a real demo, I'd move that movement update code into its own PlayerMovementSystem class, and add in a PlayerControl component. But also if you're just throwing something together, you aren't beholden to just ECS. For really simple demos I could see also just not really emphasizing ECS, but then for more advanced ones showing how to work with it. Speaking of ECS...

**Components -> Systems**

This is probably the main area Arch comes in, and leverages a nice querying/filtering API to act on Entities with the correct components in a (basically) non-allocated way:

```c#
var query = new QueryDescription().WithAll<Position,SpriteRenderer>();
Engine.World.Query(in query, (in Entity e, ref SpriteRenderer r, ref Position p) => {
  //act on entites that match the query
}
```

For Dinghy, queries are simple classes that wrap Arch queries. They are registered with the engine internals and execute based on the Interface they implement:

```c#
public abstract class RenderSystem : DSystem, IUpdateSystem {
    public void Update() {
        Render();
    }
    protected abstract void Render();
}

public class SpriteRenderSystem : RenderSystem {
    QueryDescription query = new QueryDescription().WithAll<Position,SpriteRenderer>();
    protected override void Render() {
        Engine.World.Query(in query, (in Entity e, ref SpriteRenderer r, ref Position p) => {
            //render code
        }
    }
}
```

Right now there is just IPreUpdate/IUpdate/IPostUpdate, but I may add more.

If you want to make a custom system in Dinghy, you just extend the base system and then add the interface in order to hook into standard system ticks.

**Performance**

I ran a naïve BunnyMark test and this current configuration gets me 50,000 entities all updating at 60fps. Not bad! There are lots of ways to make it faster, but it's nice to know that if you aren't thinking about performance at all you can get a pretty nice baseline without minimal setup.

## Wrap

In general, I'm really happy with where Dinghy is at and where it's going. I feel like I could honestly start giving it out to some people soon if they're interested in giving it a spin and giving me some early feedback.

If you think this is you, feel free to respond to this email! Would happy to know what you're looking for and what you want to try and make. If you aren't getting this through email, you can email me at kyle@afterschool.studio and say you're interested in trying it out!

Otherwise, hoping to have at least one more update post by the end of the year, but until then thanks for reading and being part of the ride. Talk soon!



