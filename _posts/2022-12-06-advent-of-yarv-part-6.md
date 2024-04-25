---
layout: post
title: Advent of YARV
subtitle: Part 6 - Calling methods (1)
meta:
  "twitter:card": summary
  "twitter:title": "Advent of YARV: Part 6"
  "twitter:description": "This post is part of a series about how the YARV virtual machine works."
  "twitter:site": "@kddnewton"
  "twitter:image": https://upload.wikimedia.org/wikipedia/commons/thumb/7/73/Ruby_logo.svg/1200px-Ruby_logo.svg.png
---

This blog series is about how the CRuby virtual machine works. If you're new to the series, I recommend starting from [the beginning](/2022/11/30/advent-of-yarv-part-0). This post is the first of two posts about calling methods.

Method calls are everywhere in Ruby. Even things that don't look like method calls are method calls. The following are all examples:

<table class="table-examples">
  <thead>
    <tr>
      <th>Source</th>
      <th>Receiver</th>
      <th>Method</th>
      <th>Arguments</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">self</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo?</code></td>
      <td><code class="language-ruby highlighter-rouge">self</code></td>
      <td><code class="language-ruby highlighter-rouge">foo?</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo!</code></td>
      <td><code class="language-ruby highlighter-rouge">self</code></td>
      <td><code class="language-ruby highlighter-rouge">foo!</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo.()</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">call</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo[bar]</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
      <td><code class="language-ruby highlighter-rouge">[bar]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo[bar] = baz</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">[]=</code></td>
      <td><code class="language-ruby highlighter-rouge">[bar, baz]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo bar</code></td>
      <td><code class="language-ruby highlighter-rouge">self</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">[bar]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo.bar</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">bar</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo::bar</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">bar</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo&.bar</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">bar</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo + bar</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">+</code></td>
      <td><code class="language-ruby highlighter-rouge">[bar]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo += bar</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">+</code></td>
      <td><code class="language-ruby highlighter-rouge">[bar]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">foo << bar</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge"><<</code></td>
      <td><code class="language-ruby highlighter-rouge">[bar]</code></td>
    </tr>
    <tr>
      <td><code class="language-ruby highlighter-rouge">!foo</code></td>
      <td><code class="language-ruby highlighter-rouge">foo</code></td>
      <td><code class="language-ruby highlighter-rouge">!</code></td>
      <td><code class="language-ruby highlighter-rouge">[]</code></td>
    </tr>
  </tbody>
</table>

... and many more. Every one of the examples above maps to the `send` instruction[^1], which is what we will talk about today. There is a lot of material to cover here, and quite a few concepts to introduce. Bear with me while we slog through the technical details, and you'll be rewarded with some interesting insights into Ruby's internals. You'll also get some nice diagrams at the end of the post to help elucidate everything you just learned. Let's get started. 

## Structure

First, let's talk about the structure of the instruction. The first operand is a call data structure (which we introduced in yesterday's post). It contains all of the information about the call site like the method name, number of arguments, various boolean flags, and keyword arguments.

The second operand is an optional pointer to an instruction sequence that corresponds to a block. If a block was given at the call site, the operand will be present. If it was not, the operand will be `nil`. This is the first instruction we've seen that potentially accepts a pointer to an instruction sequence as an operand, but it will not be the last.

Accepting an instruction sequence as an operand immediately indicates a couple of things. The first is that when this instruction is executed, either:

* a new frame will be pushed immediately and the instructions within the instruction sequence will be executed within that frame (as in the `send` instruction), or
* the instruction sequence will be associated with some object and executed later (as in the `definemethod` instruction).

The other thing that this indicates is that the instruction will handle the stack pointer in a special way. In every instruction that we've seen so far in this blog series (except the `leave` instruction), the stack pointer has been handled in the same way: the instruction pops the operands off the stack, performs some operation, and pushes the result back onto the stack. This is the default behavior for instructions that do not accept an instruction sequence as an operand.[^2] For instructions that do accept one, it is expected that they will manually manipulate the stack pointer in some way.

## Method types

When you're calling a method, the first thing the VM is going to do it look the method up. It does this by starting at the singleton class of the receiver and walking up the inheritance chain until it finds a method with the same name as the one you're calling. If it doesn't find one, it will then do the same walk looking for the definition of `method_missing`. `method_missing` is defined on `BasicObject` which is the top of the inheritance chain, so this is always guaranteed to find a method.[^3]

Once the method has been found, the VM is going to look up the method type.[^4] Each method type has a specific calling convention that the VM will use to call the method, so the VM needs this information to know how to set up the call. We're not going to discuss every method type in this post (actually we're only going to talk about one of them), but for completeness, here is the list of all of the different kinds of methods that the VM can call:

VM_METHOD_TYPE_ISEQ
: methods defined in Ruby

VM_METHOD_TYPE_CFUNC
: methods defined in C

VM_METHOD_TYPE_ATTRSET
: instance variable writer methods defined through `attr_writer` or `attr_accessor`

VM_METHOD_TYPE_IVAR
: instance variable reader methods defined through `attr_reader` or `attr_accessor`

VM_METHOD_TYPE_BMETHOD
: methods defined through a block (e.g., through the `define_method` method)

VM_METHOD_TYPE_ZSUPER
: method calls to `super` that forward all arguments

VM_METHOD_TYPE_ALIAS
: methods that are aliases of other methods

VM_METHOD_TYPE_UNDEF
: methods that have been undefined

VM_METHOD_TYPE_NOTIMPLEMENTED
: methods that are not implemented on account of the system they were compiled on not providing the necessary functionality

VM_METHOD_TYPE_OPTIMIZED
: internally optimized methods like `Kernel#send` and `Proc#call`

VM_METHOD_TYPE_MISSING
: the `method_missing` method

VM_METHOD_TYPE_REFINED
: methods that are refined due to refinements and scope

Once the method has been found, the visibility of the method will be checked. If the method is allowed to be called by the caller, then the next step is to set up the arguments and frame to call the method. This is where the method type comes into play. For our purposes, we will only discuss the `VM_METHOD_TYPE_ISEQ` method type today, which is the method type for methods defined in Ruby.

## Parameters

We need to make a quick distinction here before going on. In this series, we're going to use the term "arguments" to indicate values that are being passed to methods, and "parameters" to mean the names of those values as defined by the method definition. The job of the VM at this point is to take the arguments and set them up in the way that the called method expects them to be on the stack. This depends on the definition of the parameters.

Like methods, there are many kinds of parameters in Ruby. Each one has different expectations about the stack and impacts how the arguments are set up. Today, we will only be discussing the first of them, but for completeness here are all the kinds of parameters that can be defined in Ruby:

Required
: positional parameters that are defined at the beginning of the parameter list and have no optional value (e.g., `def foo(bar)`)

Required (destructured)
: positional parameters that are defined at the beginning of the parameter list and have no optional value, but are destructured into multiple variables (e.g., `def foo((bar, baz))`)

Optional
: positional parameters that have an optional value (e.g., `def foo(bar = 1)`)

Rest
: positional parameters that take the rest of the arguments using the `*` operator (e.g., `def foo(*bar)`)

Post
: positional parameters that are defined after the rest parameter (e.g., `def foo(*bar, baz)`)

Post (destructured)
: positional parameters that are defined after the rest parameter and are destructured into multiple variables (e.g., `def foo(*bar, (baz, qux))`)

Keyword
: required keyword parameters (e.g., `def foo(bar:)`)

Optional keyword (static value)
: optional keyword parameters whose default value is the same every time (e.g., an integer) (e.g., `def foo(bar: 1)`)

Optional keyword (dynamic value)
: optional keyword parameters whose default value is can change (e.g., a method call) (e.g., `def foo(bar: baz)`)

Keyword rest
: keyword parameters that take the rest of the arguments using the `**` operator (e.g., `def foo(**bar)`)

Block
: a block parameter (e.g., `def foo(&bar)`)

Forwarding
: a forwarding parameter (e.g., `def foo(...)`)

Today we will be talking exclusively about the first and simplest of these types: required parameters.

## Pushing frames

Now that we've got our method and checked its visibility, it's time to actually call it. Provided it only has required parameters, the expectation is that the order of the stack will be from bottom to top:

* receiver - the object that the method is being called on
* arguments - the arguments that are being passed to the method

The VM is going to first push a new `method` frame onto the frame stack for the method that is being called. The frame contains a pointer to the instruction sequence associated with the method definition. The VM then lowers the stack pointer for the current frame to be just below the receiver of the method. This ensures that when the `leave` instruction is executed and it is writing the return value back to the stack, it will write into the slot currently occupied by the receiver.

This is the special way of handling the stack pointer that we discussed earlier. It's important that the values aren't entirely popped off the stack until after the frame has been executed, since the arguments still need to be available to the method.

The instruction sequence for the method is then evaluated. Arguments to the method are now effectively invisible to the calling frame because the stack pointer points to a slot lower on the stack. This allows the callee frame to treat them as if they were local variables. Once the `leave` instruction is executed, the return value is written back to the stack, the frame is popped off the frame stack, and the `send` instruction finishes executing.

## Example

That was a lot of information in just text. Let's look at some diagrams to help illustrate the concepts we just discussed. If you recall from the previous posts, in all of the examples where we showed the stack, we also included an arrow to the next empty slot. This arrow is actually the stack pointer for the top frame. In those examples, we omitted the environment pointer, but we'll include it here now. Let's take a look at a very contrived example:

```ruby
def add32(value)
  value + 32
end

def celsius2fahrenheit(value)
  factor = 1.8
  add32(value * factor)
end

celsius2fahrenheit(100)
```

Let's walk through what the value stack and frame stack look like up to the point where we make the first method call (to `celsius2fahrenheit`). First, let's disassemble just the top-level instruction sequence so that we can see the instructions. Don't worry about the implementation of the instructions we don't know yet, we're going to gloss over a couple of details and come back later in the series once we've seen them.

```
== disasm: #<ISeq:<main>@test.rb:1 (1,0)-(10,23)> (catch: false)
0000 definemethod                           :add32, add32             (   1)[Li]
0003 definemethod                           :celsius2fahrenheit, celsius2fahrenheit(   5)[Li]
0006 putself                                                          (  10)[Li]
0007 putobject                              100
0009 send                                   <calldata!mid:celsius2fahrenheit, argc:1, FCALL|ARGS_SIMPLE>, nil
0012 leave
```

Now that we've disassembled, let's trace the frame and value stacks through the execution of this instruction sequence right up to the point where the first method is called.

<div align="center">
  <img src="/assets/aoy/part6-step1.svg" alt="Stacks up to the first method call">
</div>

The left column is the frame stack. The right column is the value stack. Notice that the `<main>` frame has two pointers. The one on top is the stack pointer for the next slot to write to. The one on the bottom is the environment pointer representing the base of the frame. We've made the pointers point _between_ slots on the stack because semantically they're pointing to an offset. If you were to write to one of the pointers, it would overwrite the slot just above it.

Next, we're going to execute the `send` instruction. First, this is going to pop the receiver and arguments off the stack (by changing where the stack pointer is). Then, this will push a frame onto the frame stack when we call `celsius2fahrenheit`. The instruction sequence for that method will then be executed. Let's disassemble it first:

```ruby
== disasm: #<ISeq:celsius2fahrenheit@test.rb:5 (5,0)-(8,3)> (catch: false)
local table (size: 2, argc: 1 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 2] value@0<Arg>[ 1] factor@1
0000 putobject                              1.8                       (   6)[LiCa]
0002 setlocal                               factor@1, 0
0005 putself                                                          (   7)[Li]
0006 getlocal                               value@0, 0
0009 getlocal                               factor@1, 0
0012 send                                   <calldata!mid:*, argc:1, ARGS_SIMPLE>, nil
0015 send                                   <calldata!mid:add32, argc:1, FCALL|ARGS_SIMPLE>, nil
0018 leave                                                            (   8)[Re]
```

Now let's continue our diagrams up to the point where `add32` is called.

<div align="center">
  <img src="/assets/aoy/part6-step2.svg" alt="Stacks up to the second method call">
</div>

At this point we have two frames on the frame stack, and our value stack is all set up to call `add32` via the `send` instruction. First, let's disassemble that method to see the instructions.

```
== disasm: #<ISeq:add32@test.rb:1 (1,0)-(3,3)> (catch: false)
local table (size: 1, argc: 1 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 1] value@0<Arg>
0000 getlocal                               value@0, 0                (   2)[LiCa]
0003 putobject                              32
0005 send                                   <calldata!mid:+, argc:1, ARGS_SIMPLE>, nil
0008 leave                                                            (   3)[Re]
```

Next, let's walk through the modifications to the frame and value stack as we execute the instructions for `add32` just up to the point where the `leave` instruction will be executed.

<div align="center">
  <img src="/assets/aoy/part6-step3.svg" alt="Stacks before the first leave">
</div>

At this point, we're about to execute our first `leave` instruction. This will pop the `add32` frame off the frame stack, pop the return value off the value stack, and write the return value to the parent frame's stack pointer. Below is an illustration that shows the frame and value stacks after that `leave` instruction has been executed.

<div align="center">
  <img src="/assets/aoy/part6-step4.svg" alt="Stacks after the first leave">
</div>

Notice that this actually _increased_ the stack pointer of the parent frame. This is why the `leave` instruction is said to both push and pop a value from the stack. It does, it's just not at the same point. Now let's execute the second leave.

<div align="center">
  <img src="/assets/aoy/part6-step5.svg" alt="Stacks after the second leave">
</div>

Again, the parent frame (in this case the `<main>` frame) has its stack pointer increased. This is because the `leave` instruction is pushing a value of `212` onto the stack. The last instruction to execute is the final `leave` instruction, which finishes the execution of the program.

## Wrapping up

In this post, we've seen how the frame and value stacks work together to execute method calls using the `send` instruction. We discussed only a small subset of the combination of methods and parameters (methods defined in Ruby with required parameters), but laid the foundation for the other kinds of parameters we'll see in the future. Some things to remember from this post:

* Different types of methods assume different things about the state of the stack at the point that they are called. It's the VM's responsibility to set everything up correctly before calling a method to make these assumptions valid.
* Frames keep two pointers to stack offsets around at all times: the stack pointer and the environment pointer. When a method is invoked, the parent frame's stack pointer is moved down the stack such that when the child frame is done executing the receiver of the method is overwritten by the return value.

Next time, we'll take a look at all of the different specializations of the `send` instruction.

---

[^1]: Because the `send` instruction is so prevalent, there are many specializations of it. We'll talk about all of those in the next post. In the meantime, we'll pretend we've turned off all optimizations.
[^2]: In CRuby you can find an attribute pragma in `insns.def` called `handles_sp` that indicates whether an instruction handles the stack pointer itself versus the default behavior. This attribute defaults to `true` if the instruction can have an instruction sequence as an operand, like the `send` instruction.
[^3]: There are myriad inline caches and fast paths that can be taken here to optimize method lookup, but we're ignoring those for the purposes of this post. The implementation of those alone could be an entire other series.
[^4]: In YARV, the method type corresponds to the `rb_method_type_t` enum.
