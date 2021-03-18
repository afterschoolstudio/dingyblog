---
title:  Experts and Beginners
date: 2021-03-18
tags: ["Thoughts","API Design"]
slug: experts-and-beginners

---

Something I’ve been thinking about recently with Dinghy is the idea of cognitive load. Cognitive load is basically how much information your brain is able to process at once, or over some period of time.

There’s a million ways to talk about this, but one of the most salient points I keep coming back to is one that feels tangential, or even unrelated, which is the idea of something being “beginner friendly” or being “flexible”.

I say "flexible", because we don’t really call things “expert friendly.” *Experts* like "flexibility", and flexibility is something that beginners can lose themselves inside of. However, I think this impulse is two sides of the same coin, in that both groups effectively want the same thing: the tool to bend to their needs with as little friction as possible. Said differently, they want the *cognitive load of the tool to be minimal*.

Imagine you have some coding problem. For a beginner it could be something like "How do I make a sprite appear on screen?" For an expert, if could be "How can I make many sprites appear on screen as fast as possible?" (I think there's a whole post worth of thoughts on how the only thing that may actually differentiate beginner / expert users in programming is how much they care about performance, but I'll have to save that for later). The problem, stated plainly, could be seen to have a *resolution,* (like a computer screen) that dicates not only the *clarity* of the problem, i.e. "how easy is it for me to understand that I have a problem/need?" but also the path to a given *solution*.

What complicates arriving at the solution, even at the moment the problem is percieved, is *noise.* When it comes to experts vs. beginners, I think this is the *primary* difference between them — that the *character* of the noise is different, but the *amount* is actually the same. 

Consider the following line of code:

```c#
var x = new Sprite(y);
```

A beginner might ask:

* What's var?
* What's a Sprite?
* Why is there a semi-colon?
* What are the parenthesis after Sprite?

And expert might ask:

* Is var dynamic or known at compile time?
* Do I need to cleanup the construction?
* Is X a copy of Y?
* Do I need to do further init on X after construction?
* Can Y be null?

And considering those two questions, I *really* do belive that both groups of people are experiencing the same amount of cognitive load when facing the same line. *Experts* don't have "more room for more load," they just expand the scope of the load (many of the expert questions has implications outside of just the single line). There's some sort of [Parkinson's Law](https://en.wikipedia.org/wiki/Parkinson%27s_law) for cogntive load that effectively states that "you are always at your cognitive load capacity, no matter how much you know"

---

So where does that leave tool designers? If we're not able to ever *really* decrease the cognitve load of something, what exactly *are* we doing? Especially as a beginner shifts more into expert territory — you don't really ever have "expert beginners".

I... don't really have an answer! One thing I do think is that, though the character of the noise shifts based on a given user's knowledge, the *amount* of noise if something somewhat in control of the tool creator.

In the examples above, many of the beginner's questions have much to do with C# (or whatever language) itself, whereas the expert asks questions meant to try to figure out more about the library. If we assume that someone's C# knowledge will go up over time, there's probably some middle ground where a library, seperate from C# starts to reveal itself to a beginner.

I think it's this moment I'm really focused on, and calls back a bit to the notion of Dinghy being something "cozy". In this moment, it should feel like Dinghy is sort of "holding" you — you can color inside the lines a feel like you're making meaningful progress, vs. needing to learn to draw first.

For the expert side of this, that holding can easily come from things like effective documentation so you very much know *what* you're doing *when*. There's a sort of buoying that happens for both beginners and experts here — beginners can *explore* and experts can quickly target their needs (ideally inside an IDE).

It's hard to talk about this in the abstract, so consider something Unity does that I don't think does a good job of this "holding":

```c#
this.AddComponent<SpriteRenderer>();
```

Inside Unity, if you call this function, a ```SpriteRenderer``` component is added to the GameObject you call it from. That makes sense. However, this function doesn't do the most important thing, which is actually assign a sprite to the renderer. *You have to know* that you need to do an additional call to the renderer to assign the sprite:

```c#
var renderer = AddComponent<SpriteRenderer>(); //returns the renderer
renderer.Sprite = mySprite; //assign sprite
```

This is frustrating to beginners because *nothing* about the first line implied that setup "wasn't done". This is frustrating to the expert because you know what you want to do for the component but have to have another line just to do something basic. You *could* do this:

```c#
this.AddComponent<SpriteRenderer>().sprite = mySprite;
```

But then you're unable to set any other values on the ```SpriteRenderer```.

What does a "holding" look like here? What's the alternative?

It sounds dumb, but why not just use constructors?

```c#
this.AddComponent(new SpriteRenderer(
  	sprite = mySprite;
));
```

The `sprite` parameter of `SpriteRenderer` could be required, or we could even just have a default sprite available when no sprite is provided so it "just works". This would simplify the code to:

```c#
this.AddComponent(new SpriteRenderer());
```

Calling that code would "just work". Maybe you don't care about the actual sprite in that moment anyways — you're just trying to test some movement code and want to see something on screen. If you *need* a Sprite, then all of a sudden you're having to figure out how to load a sprite into the game and now you're looking at asset pipelines and formats and... what were we doing again?

Small things like this add up and break the flow of creation. The best way I think about it is this idea of concentric circles. *Most* game frameworks / libraries really enforce the notion of "always writing the best code". This is great in some ways, but sometimes I want to write garbage code. I want to test something out, make a *sketch*. I can come back in *later* and make the code more robust if I need to, but I don't want to feel like I have to be a software architect every time I sit down to make something *dumb*. As we move to large and larger circles, we fill in more of the omitted details *as needed*, but not *by default*.

Let me be lazy, you know? Let me be chill. This is chill:

```c#
Add(new Sprite());
```

This is not chill:

```c#
var renderer = AddComponent<SpriteRenderer>();
renderer.Sprite = mySprite;
renderer.X = Screen.width / 2;
renderer.Y = Screen.height / 2;
```

Experts are always capable of writing the latter, but sometimes want to just write the former. Beginners love to write the former, and can grow to write the latter if they need to (they may not!). What's nice about the the former, from an expert perspective, is that the line also seems "simple" — it doesn't *read* like it's doing a lot of different things. You may even *expect* that it's inefficient, more so than the latter, just because the latter looks more like Real Code. However, you're probably equally suspicious of both, so why not just provide the quick and fast option!

---

This is where a lot of *the work* for Dinghy resides, at least outside of the technical parts. I’m thinking again of Kate Compton’s notion of “unfolding” complexity — with the former examples, how can we have a simple API that "just works" but also provides deeper features when needed? Here's some thoughts:

### Default Parameters

C#'s ability to provide not only default parameters for constructors, but also the ability to declare optionals means that we can provide complex defaults that "just work", and allow a user to piecemeal override them.

### Overloaded Functions

Similar to the option above, simple providing overloads for a method to encapsulate given functionality gets an API "simple" and moves complexity to the point of invocation instead of needing to know lots of random function names.

### Alternate creation routes

What if there are just two APIs for a given feature? One can be the default "easy" one, and one can be namespaced under something like Advanced. 

### Emphasize Semantic Sugar

There's some bigger concept of here of "leading by example", but leaning on semantic sugar in examples as much as possible is *nice*, often in that it has a function of focusing the problem, or at least being able to better move complexity away from the moment of writing code.

---

Dinghy will probably use some of these qualites outlined about, but at the same time I'm sure I'll discover the *Why* for a lot of things I don't particularly like and may need to bend my goals a little bit. I'm hoping though that I can still keep a lot of this in hand throughout the design process and bring it to each decision about how *you* work with Dinghy.

Until then, hope you enjoyed this post! I've got something related I'm working on that I'd like to have up next week, which is sort of a "target API usage" post that outlines what a Dinghy game should look/feel like.

---

A final note:

It’s funny talking about this because, especially considering the stack I talk about last post, this is only the tippy-top of the toolchain. Everything above I'm talking about only *really* applies to the single layer I'm working on to expose to users, but directly below that layer there's a whole chunk of indifference. I want to bring a strongly philosophy to my layer, but I think it's funny that that is built on other tech that could care less.
