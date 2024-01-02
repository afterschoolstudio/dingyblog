---
title: Dinghy Year End 2023
date: 2024-01-02
tags: ["ImGui", "Zig", "Scenes", "Demos"]
slug: dinghy-year-end-2023
---

As 2023 comes to a close I'm really satisfied with where I've been able to bring Dinghy too. I'm over the hump of the initial startup cost of just figuring out how to _do_ everything, and given it's trim, code-only nature, it means that working on it now is often just a matter of pulling together threads that already exist in the development workflow/API, adding a little bit new, and then having tangible progress.

The place this manifests most is through the current driving paradigm of Dinghy's development — demo samples. I think this is most inspired by [HaxeFlixel's demos page](https://haxeflixel.com/demos/) but also especially [Heaps' demos page](https://heaps.io/samples/) and [Sokol's samples page](https://floooh.github.io/sokol-html5/). The idea is simple — enumerate major functionality through single purpose demos. The demos act not only as API examples, but also help to teach "ergonomics" around the engine/API and show how they are all intended to be used together. Developing them also acts as nice little coding exercises that see you actually "using" the engine, and allowing you to question your own biases around the development process and tweak the experience of working with the engine to better approach the target "feeling".

Currently, there are 14 demos in Dinghy that represent core functionality:

1. Texture - Shows how to load in a texture and show it on screen
2. Texture Frame - Shows how to load in a texture and show only a portion of it ("Frame") on screen.
3. Animation - Shows how to load in a texture and slice it for animation playback.
4. Simple Update - Basic demo showing how to use an update loop.
5. Interaction - Shows how to tie into the input system for feedback
6. Bunnymark - Smoke test benchmark
7. Physics - Shows how to setup entities for use in a physics system (uses Volatile physics)
8. Shape - Shows how to spawn a simple shape for an entity (basically a textureless GL_Primitive type object)
9. Physics Shape - same as physics demo but uses shapes
10. Entity Emitter - Just shows how to spawn lots of entities
11. Collision - Shows how to do things like mouseover for entities as well as find closest contact point, etc.
12. Grid - Shows example of using the grid class to spawn entities in a user-specific grid
13. Particle System - Demonstrates the particle system in the engine

At first, each demo was just a function in a top level program, and running them was a matter of commenting/uncommenting the specific demo and rebuilding the project.

This was okay at first, but as the number of demos started to climb it became a bit unwieldy. You can still write Dinghy games fully in a single file (and I recommend you do so for small stuff!) but I realized I needed a better organizational scheme. Additionally, I wanted to be able to switch demos at runtime, which also meant the ability to setup/teardown demos as well as the interface for navigating between them.

These two constraints led the two other major efforts of the past few months since the last update —the creation of a scene system and integration with Dear Imgui.

Before I got into the specifics, I’ll show the end result. Here's me just navigating through the demos one by one in Dinghy:

https://youtu.be/rgyOkpba4ME

Pretty nice!

The scene system is inspired a bit by [Ceramic’s scene system](https://ceramic-engine.com/guides/scenes/), and works in the ECS like anything else. This was definitely a big "rubber meets the road" moment as well for deciding if Dinghy was going to truly be only code. There are obvious ways for scenes to just be data and have them configured at runtime a la Unity, but instead of going down that path I wanted to maintain that Dinghy was code-only and as such chose a more OOP model for Scenes (instead of data). You could still totally build a runtime scene editor to plug into the scene system, but out of the box I’m providing a strongly typed object and some utility functions to configure scenes and their lifecycles. Here's a code example of a scene (the basic Texture demo):

```c#
namespace Dinghy.Sandbox.Demos;

[DemoScene("01 Texture")]
public class Texture : Scene
{
    private TextureData conscriptImage;
    private SpriteData fullConscript;
    public override void Preload()
    {
        conscriptImage = new TextureData("conscript.png");
        fullConscript = new(conscriptImage);
    }

    public override void Create()
    {
        Engine.SetTargetScene(this);
        new Sprite(fullConscript);
    }
}
```

Scenes also do provide Update hooks for objects (the SimpleUpdate example):

```c#
using Dinghy.Core;

namespace Dinghy.Sandbox.Demos;

[DemoScene("04 Simple Update")]
public class SimpleUpdate : Scene
{
    private TextureData conscriptImage;
    private SpriteData conscriptFrame0;
    public override void Preload()
    {
        conscriptImage = new TextureData("conscript.png");
        conscriptFrame0 = new(conscriptImage, new Frame(0,0,64,64));
    }

    Sprite e;
    Point startPos = (0,0);
    public override void Create()
    {
        Engine.SetTargetScene(this);
        startPos = ((Engine.Width / 2f) - 32, (Engine.Height / 2f) - 32);
        e = new Sprite(conscriptFrame0)
        {
            X = (int)startPos.X,
            Y = (int)startPos.Y,
        };
    }

    public override void Update(double dt)
    {
        e.X = (int)startPos.X + (int)(Math.Sin(Engine.Time) * 100);
        e.Rotation = (float)Engine.Time;
        var scale = (Math.Cos(Engine.Time) * 2);
        e.ScaleX = (float)scale;
        e.ScaleY = (float)scale;
    }
}
```

Scenes are meant as a better organization system for units of functionality in a game, and multiple can be loaded at once. Like Unity, you do set an "Active" scene (Dinghy calls it a "Target" scene) that designates where spawned entities are spawned into. Entity constructors *can* override this functionality and spawn entities into non-target scenes as well, but generally you probably want to set a target scene.

Scenes were one of those things that were conceptually easy to do based on all the previous work, so it was exciting to get to work on this because it felt like I was really reaping what I had sown (in a good way).

Getting Dear ImGui working was a bit harder because it meant threading the needle between Dear ImGui, Sokol, cimgui.h, and the Zig build pipeline. Specifically, it was weird because I was mixing C and C++ libs together in a target final DLL. What was nice to find was that the process was a bit simpler to do in the end than I thought it was, but at the same time Zig is still very much in development with pretty bad docs, so finding answers for questions I had was a matter of pestering the Zig Discord until I got the answer I was looking for. For reference, here's my `build.zig` file, with it largely being an edit of the [one Andres created for the Sokol Zig bindings](https://github.com/floooh/sokol-zig/blob/master/build.zig):

```zig
const std = @import("std");
const builtin = @import("builtin");
const Builder = std.build.Builder;
const LibExeObjStep = std.build.LibExeObjStep;
const CrossTarget = std.zig.CrossTarget;
const Mode = std.builtin.Mode;

pub const Backend = enum {
    auto,   // Windows: D3D11, macOS/iOS: Metal, otherwise: GL
    d3d11,
    metal,
    gl,
    gles2,
    gles3,
    wgpu,
};

pub const Config = struct {
    backend: Backend = .auto,
    force_egl: bool = false,

    enable_x11: bool = true,
    enable_wayland: bool = false
};

pub fn build(b: *std.Build) void {

    var config: Config = .{};
        
    const force_gl = b.option(bool, "gl", "Force GL backend") orelse false;
    config.backend = if (force_gl) .gl else .auto;
    config.enable_wayland = b.option(bool, "wayland", "Compile with wayland-support (default: false)") orelse false;
    config.enable_x11 = b.option(bool, "x11", "Compile with x11-support (default: true)") orelse true;
    config.force_egl = b.option(bool, "egl", "Use EGL instead of GLX if possible (default: false)") orelse false;

    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    
    const cpp_sources = [_][]const u8 {
        "imgui.cpp",
        "../../cimgui/src/cimgui/cimgui.cpp",
        "../../cimgui/src/cimgui/imgui/imgui.cpp",
        "../../cimgui/src/cimgui/imgui/imgui_demo.cpp",
        "../../cimgui/src/cimgui/imgui/imgui_draw.cpp",
        "../../cimgui/src/cimgui/imgui/imgui_tables.cpp",
        "../../cimgui/src/cimgui/imgui/imgui_widgets.cpp",
    };
    
    const c_sources = [_][]const u8 {
        "sokol.c"
    };
    
     const dll = b.addSharedLibrary(.{
        .name = "sokol",
        .target = target,
        .optimize = optimize
    });
    dll.linkLibCpp();
    dll.linkLibC();
    
    var _backend = config.backend;
    if (_backend == .auto) {
        if (dll.target.isDarwin()) { _backend = .metal; }
        else if (dll.target.isWindows()) { _backend = .d3d11; }
        else { _backend = .gl; }
    }
    const backend_option = switch (_backend) {
        .d3d11 => "-DSOKOL_D3D11",
        .metal => "-DSOKOL_METAL",
        .gl => "-DSOKOL_GLCORE33",
        .gles2 => "-DSOKOL_GLES2",
        .gles3 => "-DSOKOL_GLES3",
        .wgpu => "-DSOKOL_WGPU",
        else => unreachable,
    };

    if (dll.target.isDarwin()) {
        inline for (c_sources) |csrc| {
            dll.addCSourceFile(.{
                .file = .{ .path = csrc },
                .flags = &[_][]const u8{ "-ObjC", "-DIMPL", backend_option },
            });
        }
        inline for (cpp_sources) |csrc| {
            dll.addCSourceFile(.{
                .file = .{ .path = csrc },
                .flags = &[_][]const u8{ "-ObjC++", "-DIMPL", backend_option },
            });
        }
        dll.linkFramework("Cocoa");
        dll.linkFramework("QuartzCore");
        dll.linkFramework("AudioToolbox");
        if (.metal == _backend) {
            dll.linkFramework("MetalKit");
            dll.linkFramework("Metal");
        }
        else {
            dll.linkFramework("OpenGL");
        }
    } else {
        var egl_flag = if (config.force_egl) "-DSOKOL_FORCE_EGL " else "";
        var x11_flag = if (!config.enable_x11) "-DSOKOL_DISABLE_X11 " else "";
        var wayland_flag = if (!config.enable_wayland) "-DSOKOL_DISABLE_WAYLAND" else "";

        inline for (c_sources) |csrc| {
            dll.addCSourceFile(.{
                .file = .{ .path = csrc },
                .flags = &[_][]const u8{ "-DIMPL", backend_option, egl_flag, x11_flag, wayland_flag },
            });
        }
        inline for (cpp_sources) |csrc| {
            dll.addCSourceFile(.{
                .file = .{ .path = csrc },
                .flags = &[_][]const u8{ "-DIMPL", backend_option, egl_flag, x11_flag, wayland_flag },
            });
        }

        if (dll.target.isLinux()) {
            var link_egl = config.force_egl or config.enable_wayland;
            var egl_ensured = (config.force_egl and config.enable_x11) or config.enable_wayland;

            dll.linkSystemLibrary("asound");

            if (.gles2 == _backend) {
                dll.linkSystemLibrary("glesv2");
                if (!egl_ensured) {
                    @panic("GLES2 in Linux only available with Config.force_egl and/or Wayland");
                }
            } else {
                dll.linkSystemLibrary("GL");
            }
            if (config.enable_x11) {
                dll.linkSystemLibrary("X11");
                dll.linkSystemLibrary("Xi");
                dll.linkSystemLibrary("Xcursor");
                
            }
            if (config.enable_wayland) {
                dll.linkSystemLibrary("wayland-client");
                dll.linkSystemLibrary("wayland-cursor");
                dll.linkSystemLibrary("wayland-egl");
                dll.linkSystemLibrary("xkbcommon");
                
            }
            if (link_egl) {
                dll.linkSystemLibrary("egl");
            }
        }
        else if (dll.target.isWindows()) {
            dll.linkSystemLibraryName("kernel32");
            dll.linkSystemLibraryName("user32");
            dll.linkSystemLibraryName("gdi32");
            dll.linkSystemLibraryName("ole32");
            if (.d3d11 == _backend) {
                dll.linkSystemLibraryName("d3d11");
                dll.linkSystemLibraryName("dxgi");
            }
        }
    }
    
    b.lib_dir = "../../../../Dinghy.Core/libs";
    b.installArtifact(dll);
}
```

On the actual library side, I ended up using the cimgui.h C bindings for the actual C# PInvoke code, which, as you can see above, is compiled alongside the normal C++ Dear ImGui library in the final DLL. I tried binding against the raw Dear ImGui headers, but ran into some issues with the PInvoke code for the C++ headers that were maybe solvable but I just opted for cimgui.h instead. Like everything else, the bindings are generated via [ClangSharpPInvokeGenerator](https://github.com/dotnet/ClangSharp) for MAXIMAL SPEED.

Once the scene system and Dear ImGui were setup, I set about porting all the demos to the scene system and used some nice little bit of reflection (did you notice the attributes on the Demo code samples?) to procedurally build the UI to move between them. This means the overhead to add new demos is really low, which helps me maintain development velocity and not feel too much pressure about how hard it may be to add a new one. Here's the relevant code:

```c#
ImGUIHelper.Wrappers.BeginMainMenuBar();
if (ImGUIHelper.Wrappers.BeginMenu("Dinghy")) {
  if (ImGUIHelper.Wrappers.BeginMenu("Demos")) {
    Scene? scene = null;
    foreach (var type in demoTypes) {
      if(ImGUIHelper.Wrappers.MenuItem(type.Name)) {
        scene = Util.CreateInstance(type.Type) as Scene;
        scene.Name = type.Name;
      }
    }
    if (scene != null) {
      Engine.TargetScene.Unmount(() => {
        scene.Mount(0);
        scene.Load(scene.Start);
      });
    }
    ImGUIHelper.Wrappers.EndMenu();
  }
  ImGUIHelper.Wrappers.EndMenu();
}
ImGUIHelper.Wrappers.EndMainMenuBar();
```

You can see a peek at some scene lifecycle management stuff here as well — Unmount has a callback you can tie into to do whatever else — in this case I mount a new scene and set it to Load, with a callback for the scene to start right after it loads.

You can also see some `ImGUIHelper.Wrapper` methods — working with raw PInvoke ImGui bindings is a bit onerous for end users (lots of `fixed` statements, etc.), so I'm working to wrap a lot of the functionality around managed function calls that at least get rid of boilerplate for ImGUI stuff and provide ways to pass strings to function calls without needing to explicitly call `fixed` in your own code. For example, `ImGUIHelper.Wrappers.BeginMenu("Dinghy")` wraps this way:

```c#
public static unsafe bool BeginMenu(string name, bool enabled = true)
{
    var b = System.Text.Encoding.UTF8.GetBytes(name);
    fixed (byte* ptr = b)
    {
        return Internal.Sokol.ImGUI.igBeginMenu((sbyte*)ptr,(byte)(enabled ? 1 : 0)) > 0;
    }
}
```

This way it's really easy to add in normal ImGUI calls wherever. For example (again), one thing I did that you may have seen in the particle demo is that I can use a similar reflection thing a la Unity to draw runtime property editors for any object:

{{< figure src="/images/particleeditor.png" caption="Particle Editor" >}}

This code is all you need to do that:

```c#
DrawEditGUIForObject("emitter",ref emitter);
```

You can easily do this for any object or config anything. It's really nice! I suspect this function may be used a lot for Dinghy games so I'm working on fleshing it out a lot with nice defaults based on field types, and also providing ways for devs to author their own properties for editing (think Unity CustomPropertyDrawer but with ImGUI). I'll likely defer serialization to end-users as well, but proving a nice on-demand edit option for users is pretty nice (I think!).

A note on Magic: Dinghy's core pillars are still immediacy, coziness, and magic. Right now I think I'm doing a lot in the immediacy/coziness dimension, but the engine definitely lacks the Magic. I've got plans for implementing that stuff down the road, but for now the core functionality is still the target, and the Magic is something that would build on top of that foundation. As such, not a lot of magic right now!

Lastly, one thing I've been thinking a lot about is [Luau](https://luau-lang.org). I [spiked a prototype to get Luau working in C#](https://github.com/afterschoolstudio/Luau.NET), and the more I think about it the more I think it would be really cool for most all of Dinghy's functionality to be exposed via Luau scripts. This would unlock things like live-coding at runtime, which, afaik, isn't a thing most engines offer. [I'm really inspired by what seems possible with Luau](https://www.remedygames.com/article/how-northlight-makes-alan-wake-2-shine) (not to mention it being battle-tested by the Roblox team), and I like the idea that Dinghy could, with Luau support, be a stepping stone for Roblox players/creators looking to leave the Roblox ecosystem and do standalone development. Also, live-coding is *definitely* in line with the principle of "immediacy", so I'm really considering it. That said, the only work done so far is that prototype, but expect to hear more about Luau in Dinghy in the future. 

---

In all, I'm feeling really good where Dinghy is at. It's really meeting the exact goals I set out for it from the start, and I can feel how already I want to make stuff with it. If you're someone also interested, please reach out to me! I'd love to put together a small cohort of users to start getting a sense of how people like it/what they want/need. There are a ton of loose ends and missing functionality in the engine, but I think it's already at, if not VERY close to, having a lot of what you need to make a simple and small game to post somewhere like itch. If this is you, please reach out!

Otherwise, looking forward to sharing more progress in the next few months about where things are at and also hopefully share the first real game made with it!

Until next time!