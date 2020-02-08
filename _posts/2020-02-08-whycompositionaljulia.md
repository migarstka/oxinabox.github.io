---
layout: default
title: "JuliaLang: The Ingredients for a Composable Programming Language"
tags:
    - julia
    - jupyter-notebook
---

One of the most remarkable things about the Julia programming language,
is how well the packages compose.
You can almost always reuse someone elses types or methods in your own software without issues.
This is generally taken on a high level to be true of all programming languages because that is what a library is.
However, experienced software engineers often note that its surprisingly difficult in practice to take something from one project and use it in another without tweaking.
But in the Julia Ecosystem, which was written mostly by Grad students, this seems to mostly work.
This post explores some theories on why;
and some lose recommendations to future language designers.
<!--more-->


This blog post is based on a talk I was invited to give at the 2020 [F(by) conference](fby.dev).
Hopefully videos from that will be up soon, and I will link it here.
This blog post is a bit ad-hoc in its ordering and content because of its origin as a talk, which was ad-hoc in ordering and content.
I trust the reader will forgived me.

## What do I mean by composable ?

### Examples:

 - If you want to add tracking of measurement error to a scalar number, you shouldn't have to say anything about how your new type interacts with arrays **([Measurements.jl](https://github.com/JuliaPhysics/Measurements.jl))**
 - If you have a Differential Equation solver, and a Neural Network library, then you should just be able to have neural ODEs (**[DifferentialEquations.jl](https://github.com/JuliaDiffEq/DifferentialEquations.jl) / [Flux.jl](https://github.com/FluxML/Flux.jl)**)
 - If you have a package to add names to the dimensions of an array, and one to put arays on the GPU, then you shouldn't have to write code to have named arrays on the GPU (**[NamedDims.jl](https://github.com/invenia/NamedDims.jl) / [CUArrays.jl](https://github.com/JuliaGPU/CuArrays.jl)**)

### Why Julia is it this way?

The theory I prose heremay sound counter-intuitive
I suggest, that julia code is so reusable,
are because the language has not just good features, but **weak and missing features.**

Missing features like:
 - Weak conventions about namespace polution
 - Never got round to making it easy to use local modules, outside of packages
 - A type system that can't be used to check correctness

But that these are countered by, or allow for other features:
 - Strong convention about talking to other people
 - Very easy to create packages
 - Duck-typing, and multiple dispatch, together


## Julia namespacing is used in a leaky way

Common advise when loading code form another module in most languagage communities is:
only import what you need.
e.g `using Foo: a, b c`

Common practice in Julia is to do:
`using Foo`,
which imports everything everything that the author of `Foo` marked to be **exported**.

You don't have to, but it's common.

But what happens if one has package:
 - `Bar` exporting `predict(::BarModel, data)`,
 - and another `Foo` exporting `predict(::FooModel, data)`

and one does:
```julia
using Foo
using Bar
training_data, test_data = ...
mbar = BarModel(training_data)
mfoo = FooModel(training_data)
evaluate(predict(mbar), test_data)
evaluate(predict(mfoo), test_data)
```

If you have multiple `using`s trying to bring the same name into scope,
then julia throws an error.
Since it can't work out which to use.

As a user you can tell it what to use.

```julia
evaluate(Bar.predict(mbar), test_data)
evaluate(Foo.predict(mfoo), test_data)
```

### But the package authors can solve this:
There is no name collision if both names *are* overloaded the from the same namespace.

If both `Foo` and `Bar` are overloading `StatsBase.predict` everything works.

```julia
using StatsBase  # exports predict
using Foo  # overloads `StatsBase.predict(::FooModel)
using Bar  # overloads `StatsBase.predict(::BarModel)
training_data, test_data = ...
mbar = BarModel(training_data)
mfoo = FooModel(training_data)
evaluate(predict(mbar), test_data)
evaluate(predict(mfoo), test_data)
```


### This encourages people to work together

Name collisions makes package authors to come together and create base package (like `StatsBase`) and agree on what the functions mean.

They don't have to, since the user can still solve it, but it encourages it.
Thus you get package authors thinking about other packages that might be used with theirs.

One can even overload functions from multiple namespaces if you want;
e.g. all of `MLJBase.predict`, `StatsBase.predict`, `SkLearn.predict`.
Which might all have slightly different interfaces targetting different use cases.


## Its easier to create a package than a local module.

Many languages have one module per file,
and you can load that module e.g. via
`import Filename`
from your current directory.

You can make this work in Julia also, but it is surprisingly fiddly.

What is easy however, is to create and use a package.

### What does making a local module generally give you?

 - Namespacing
 - The feeling you are doing good software engineering
 - Easier to transition later to a package

### What does making a Julia package give you?
 - All the above plus
 - Standard directory structure, `src`, `test` etc
 - Managed dependencies, both what they are, and what versions
 - Easy re-distributivity -- harder to have local state
 - Test-able using package manager's `pkg> test MyPackage`

The [recommended way](https://github.com/invenia/PkgTemplates.jl/) to create packages also ensures:

 - Continuous Integration(/s) Setup
 - Code coverage
 - Documentation setup
 - License set

### Testing Julia code is important.
Julia uses a JIT compiler, so  even compilation errors don't arive til run-time.
As a dynamic language the type system says very little about correctness.

Testing julia code is important.
If code-paths are not covered in tests their is almost nothing in the language itself to protect them from having any kind of error.

So its important to have Continuous Integration and other tooling all setup

### Making package making trival gets more packages made

Recall that many Julia package authors are Grad students who are trying to get their next paper complete.
Lots of scientific code never gets released, and lots of the code that does never gets made usable for others.
But if it is easier to make a package, than it is to not, then it does.
And once it is a package people start thinking more like package authors, and considering how it will be used.

Its not a silver bullet but its one more push in the right direction.

## Multiple Dispatch + Duck-typing

**Assume it walks like a duck and talks like a duck, and if it doesn't fix that.**

---

#### Aside: Open Classes
Another closely related factor is **Open Classes.**
But I'm not going to talk about that today, I recommend finding other resource to read on it.
Such as [Eli Bendersky's blog post on the expression problem](https://eli.thegreenplace.net/2016/the-expression-problem-and-its-solutions#is-multiple-dispatch-necessary-to-cleanly-solve-the-expression-problem)
You need to allow new methods to be added to existing classes, in the first place.
Some languages (e.g. Java) require that methods literally be places inside the same file as the class.
This means there is no way to add methods in another code-base, even unrelated ones.

---

### We would like to use some coode from a library
Consider I might have a type from the **Ducks** library.

**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
struct Duck end

walk(self) = println("🚶 Waddle")
talk(self) = println("🦆 Quack")

raise_young(self, child) = println("🐤 ➡️ 💧 Lead to water")
{% endhighlight %}
</div>

and I have some code I want to run, that I wrote:

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
function simulate_farm(adult_animals, baby_animals)
    for animal in adult_animals
        walk(animal)
        talk(animal)
    end

    # choose the first adult and make it the parent for all the baby_animals
    parent = first(adult_animals)
    for child in baby_animals
        raise_young(parent, child)
    end
end
{% endhighlight %}
</div>

#### Lets give it a try:

3 Adult ducks, 2 Baby ducks:
**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
simulate_farm([Duck(), Duck(), Duck()], [Duck(), Duck()])
{% endhighlight %}
</div>

**Output:**

<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🚶 Waddle
🦆 Quack
🚶 Waddle
🦆 Quack
🚶 Waddle
🦆 Quack
🐤 ➡️ 💧 Lead to water
🐤 ➡️ 💧 Lead to water
{% endhighlight %}
</div>

Great, it works

#### Ok now I want to extend it with my own type. A Swan

**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
struct Swan end
{% endhighlight %}
</div>

Lets test with just 1 first:
<div class="jupyter-input jupyter-cell">
{% highlight julia %}
simulate_farm([Swan()], [])
{% endhighlight %}
</div>

**Output:**

<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🚶 Waddle
🦆 Quack
{% endhighlight %}
</div>

The **Waddle** was right, but Swans don't **Quack**.

We did some duck-typing -- Swans walk like ducks,
but they don't talk like ducks.


We can solve that with **single dispatch**.

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
talk(self::Swan) = println("🦢 Hiss")
{% endhighlight %}
</div>


**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
simulate_farm([Swan()], [])
{% endhighlight %}
</div>

**Output:**

<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🚶 Waddle
🦢 Hiss
{% endhighlight %}
</div>


Great, now lets try a whole farm of Swans:

**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
simulate_farm([Swan(), Swan(), Swan()], [Swan(), Swan()])
{% endhighlight %}
</div>

**Output:**
<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🚶 Waddle
🦢 Hiss
🚶 Waddle
🦢 Hiss
🚶 Waddle
🦢 Hiss
🐤 ➡️ 💧 Lead to water
🐤 ➡️ 💧 Lead to water
{% endhighlight %}
</div>

That's not right. Swans do not lead their young to water.

They carry them

![](https://p1.pxfuel.com/preview/695/84/40/nature-bird-swan-young-royalty-free-thumbnail.jpg)

Once again we can solve this with **single dispatch**.

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
raise_young(self::Swan, child::Swan) = println("🐤 ↗️ 🦢 Carry on back")
{% endhighlight %}
</div>


Trying again:

**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
simulate_farm([Swan(), Swan(), Swan()], [Swan(), Swan()])
{% endhighlight %}
</div>

**Output:**

<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🚶 Waddle
🦢 Hiss
🚶 Waddle
🦢 Hiss
🚶 Waddle
🦢 Hiss
🐤 ↗️ 🦢 Carry on back
🐤 ↗️ 🦢 Carry on back
{% endhighlight %}
</div>

#### Now I want a Farm with mixed poultry.

2 Ducks, a Swan, and 2 baby swans

**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
simulate_farm([Duck(), Duck(), Swan()], [Swan(), Swan()])
{% endhighlight %}
</div>

**Output:**

<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🚶 Waddle
🦆 Quack
🚶 Waddle
🦆 Quack
🚶 Waddle
🦢 Hiss
🐤 ➡️ 💧 Lead to water
🐤 ➡️ 💧 Lead to water
{% endhighlight %}
</div>


Thats not right again.

<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🐤 ➡️ 💧 Lead to water
{% endhighlight %}
</div>

What happened?

We had a Duck, raising a baby Swan, and it lead it to water.

If you know about raising poultry, then you will know:
_Ducks given baby Swans to raise, will just abandon them._



But how will we code this?

### Option 1: Rewrite the Duck

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
function raise_young(self::Duck, child::Any)
    if child isa Swan
        println("🐤😢 Abandon")
    else
        println("🐤 ➡️ 💧 Lead to water")
    end
end
{% endhighlight %}
</div>

#### Rewriting the Duck has problems
 - Have to edit someone elses library, to add support for *my* type.
 - This could mean adding a lot of code for them to maintain
 - Does not scale, what if other people wanted to add Chickens, Geese etc.

##### Varient: Monkey-patch
 - If the language supports monkey patching, could do it that way
 - but it means copying their code into my library, will run in to issues like not being able to update.
 - Scaled to other people adding new types even worse, since no longer a central canonical source to copy

##### Varient: could fork their code
 - That is giving up on code reuse.

### Option 2: Inherit from the Duck

(NB: this example is **not** valid julia code)
<div class="jupyter-input jupyter-cell">
{% highlight julia %}
struct DuckWithSwanSupport <: Duck end

function raise_young(self::DuckWithSwanSupport, child::Any)
    if child isa Swan
        println("🐤😢 Abandon")
    else
        raise_young(upcast(Duck, self), child)
    end
end
{% endhighlight %}
</div>

#### Inheriting from the Duck has problems:
 - Have to replace every `Duck` in my code-base with `DuckWithSwanSupport`
 - If I am using other libraries that might return a `Duck` I have to deal with that also

##### Still does not scale:
If someone else implements `DuckWithChickenSupport`, and I want to use both there code and mine, what do?
 - Inherit from both? `DuckWithChickenAndSwanSupport`
 - This is the classic multiple inheritance Diamond problem.
 - It's hard. Even in languages supporting multiple inheritance, they may not support it in a useful way for this without me writing special cases for many things.

There *are* engineering solutions around all this.
It is beyond the scope of this post to detail them.
But in sort they are more complex to use than multiple dispatch.
One can use various design patterns to emulate multiple dipatch.

### Option 3: Multiple Dispatch

This is clean and easy:

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
raise_young(parent::Duck, child::Swan) = println("🐤😢 Abandon")
{% endhighlight %}
</div>

Trying it out:

**Input:**

<div class="jupyter-input jupyter-cell">
{% highlight julia %}
simulate_farm([Duck(), Duck(), Swan()], [Swan(), Swan()])
{% endhighlight %}
</div>

**Output:**

<div class="jupyter-stream jupyter-cell">
{% highlight plaintext %}
🚶 Waddle
🦆 Quack
🚶 Waddle
🦆 Quack
🚶 Waddle
🦢 Hiss
🐤😢 Abandon
🐤😢 Abandon
{% endhighlight %}
</div>


## But does this happen in the wild?

Turns out it does.

The need to extend operations to act on new combinations of types shows up all the time in scientific computing.
I suspect it shows up more generally, but we've learned to ignore it.


If you look at a list of BLAS, methods you will see just this, encoded in the function name
E.g.
 - `SGEMM` - matrix matrix multiply
 - `SSYMM` - symmetric-matrix matrix multiply
 - ...
 - `ZHBMV` - complex hermitian-banded-matrix vector multiply

And turns out people keep wanting to make more and more matrix types.

 - Block Matrix
 - Banded Matrix
 - Block Banded Matrix (where the band is made up of blocks)
 - Banded Block Banded Matrix (where the band is made up of blocks that are themselved banded).

That is before other things you might like to to to a Matrix, which you'd like to encode in its type:
 - Running on a GPU
 - Tracking Operations for AutoDiff
 - Naming dimensions, for easy lookup
 - Distributing over a cluster

These are all important and show up in crucial applications.
When you start applying things across disciplines, they show up even more.
Like advancements in Neural Differential Equations, which needs:

 - all the types machine learning research has invented,
 - and all the types differential equation solving research has invented,

and wants to use them together.

So its not a reasonable thing for a numerical language to say that they've enumerated all the matrix types you might ever need.

## Inserting a human into the JIT

### Basic functionality of an Tracing JIT:

 - Detect important cases via tracing
 - Compile specialized methods for them

This is called specialization.

### Basic functionalitionality of Julia's JIT:

 - Specialize all methods on all types that they are called on as they are called

This is pretty good: its a reasonable assumption that the types are going to an important case.

### What does multiple dispatch add ontop of Julia's JIT?

It lets a human tell it how that specialization should be done.
Which can add a lot of information.

### Consider Matrix multiplication.

We have

- `*(::Dense, ::Dense)`:
    - multiply rows by columns and sum.
    - Takes $O(n^3)$ time


 - `*(::Dense, ::Diagonal)` or `*(::Diagonal, ::Dense)`:
    - column-wise/row-wise scaling.
    - $O(n^2)$ time.


 - `*(::OneHot, ::Dense)` or `*(::Dense, ::OneHot)`:
    - column-wise/row-wise slicing.
    - $O(n)$ time.


 - `*(::Identity, ::Dense)` or `*(::Dense, ::Identity)`:
    - no change.
    - $O(1)$ time.


### Anyone can have basic fast array processing by throwing the problem to BLAS, or a GPU.

But not everyone has Array types that are parametric on their scalar types;
and the ability to be equally fast in both.

Without this, your array code, and your scalar code can not be disentangled.

BLAS for example does not have this.
It has a unique code for each combination of scalar and matrix type.

With this seperation, one can add new scalar types:
 - Dual numbers
 - Measument Error tracking numbers
 - Symbolic Algebra numbers

Without ever having to touch array code, except as a late-stage optimization.

Otherwise, one needs to implement array support into one's scalars, to have reasonable performance at all.

## It is good to invent new languages

People need to invent new languages.
Its a good time to be inventing new languages.
It's good for the world.
Its good for me because I like cool new things.

I’ld just really like those new languages to please have:
 - multiple dispatch
 - open classes, so you can add methods to things.
 - array types that are parametric on their scalar types, at the type level
 - A package management solution built-in, that everyone uses.