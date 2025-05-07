**(this is a backup of [my post on PureScript discourse](https://discourse.purescript.org/t/psa-stop-recommending-halogen-we-have-react/4920))**

I've been meaning to write this post for ½ an year, because I think the current situation is actively hurting community, mostly by turning away new users/contributors.

# Abstract

Every source I found out there claims Halogen to be the superior framework for PureScript frontend development. Practical experience showed the reverse: Halogen is considerably weaker compared to [React Hooks](https://pursuit.purescript.org/packages/purescript-react-basic-hooks). This makes new people try Halogen out for their application, and then have larger probability to end up failing.

This text explains the experience, differences and why Halogen shouldn't be preferred over React Hooks.

# Who am I

I wasn't a web dev until recently. That is to say I don't have a bias to React as you could expect from a web programmer. My knowledge in this context was as basic as "I know there's HTML and I have to make it interact with user via JS *(or whatever compiles to it)*, and use HTTP for communication". "What's React?" — "Idk, apparently one of the many libs web devs are using".

I am an experienced programmer though, just not in this field. I have huge background in system-level programming, Linux; and at different times of my career I have coded in a dozen of programming languages. MASM, C, C++, C#, ELisp, Haskell, TLA+… — did it all. Not necessarily remember them because you forget things you're not using *(last I coded in Haskell was like 6+ years ago)*, but I do know something about maintainability and how things should work in general.

Won't dive into why a non-web dev was assigned to develop web app. In short, there were a lot of research and other stuff I had to do, web-devel just kind of came along. I looked at TypeScript, found its type-system to be pretty poor *(the "structural typing", consts not being consts, etc)* and ended up with PureScript.

Halogen ended up the framework I started writing my app in, because people have been praising its awesomeness and most tutorials are targeted towards it.

Halogen ended up almost killing my project, I kid you not 😊 Halogen was one of the major reasons for my overtime work. Hadn't I realize things are going badly and my PureScript experiment is deemed to fail, so it would. IIRC I wrote 2 web-pages in Halogen and their maintainability was pretty poor for reasons outside of my control. My evaluation of "is it the PureScript" came as "no". I mean, as of today PS is known to have a compiler that gives absolutely terrible and misleading error messages, but with enough experience you can get used to them and start seeing them through. They do annoy, but weren't the problem. Instead it were the excessive and error-prone abstractions provided by Halogen.

Amidst all the development and with deadlines looming nearer I made YOLO jump to React. Had nothing to lose anymore: the project really was a failure if I stayed on Halogen. And… I clutched it out! 😊

# Do you claim Halogen is bad?

No. I know how the text below may sound, so to put it out beforehand: Halogen is an amazing project, and I give props to all the people involved for this work. I am sure a lot of effort was put into the abstractions and making them properly connect and to enable web development.

It's just… it just works, you know. May be suitable for times when nothing else is around. People would be coming up with workarounds for various stuff, but Halogen does its job.

It just so happened that there is also React, which does same job done but better. Having coded in both frameworks, the only possible reason I imagine you might want Halogen is if you simultaneously α) have project written in it and β) other devs don't to want to learn React. The difference is big enough that in absence of β you may want to gradually migrate to React to save human-hours in the long run.

So… keep in mind the context of comparison of two frameworks.

# The comparison

React has much more verbose code for hooking up the initial component into the HTML *(the `createRoot` and `renderRoot` stuff)*, but it's only one such location which you rarely if ever read/edit. So it doesn't really matter, unless you frequently create new projects.

Basic DOM tree is written very similarly:

```haskell
import Halogen.HTML    as H
import React.Basic.DOM as R

html1 = H.div_ [ H.button_ [ H.text "Submit" ]]
html2 = R.div_ [ R.button_ [ R.text "Submit" ]]
```

## Component effects

A component *(part of a DOM tree with side-effects handling)* is where the difference shows up. In Halogen you define some `data Action = Branch1 | Branch2 | …` and then Halogen says: "every event *(like `onClick`, `onKeyDown`, etc)* will return a `BranchX` of `Action`, to be processed later in `handleAction` function". If you write a web app, however you try to break things to different components *(which is a separate issue I'll go into later)*, you will inevitably have single `handleAction` handling completely unrelated events, which becomes just messy to read.

Compared to React you'll also have a bunch of `data Action` values, which are completely unrelated to each other, and may easily become orphaned with time without you noticing. And every time you want to see the actual handler you have to go through this indirection point.

In React you just assign the callback, that's it. Easy. No additional `handleAction`, no clumping unrelated handlers under a single `data Action` umbrella… But is it type-safe…? Of course! You don't execute an effect handler inside pure code, nothing like that. You just create a callback, which will be called by some other code at some point of time.

This whole `data Action` indirection creates no benefits compared to a callback; what it does create though is a burden for you to maintain.

## Refs

"Ref"s are basically a way to refer to a DOM element without explicit effects. So like, wasn't it for "ref"s, you'd have to explicitly assign an `id` to an element and then effectfully `getElementById()`.

In React `ref` acts like a pipe: you create it in the component body, and then you pass it to the element and to the action. Handy. One could argue there is effect *(as the ref content gets changed)*, but it's hidden.

In Halogen it isn't a pipe. Instead `RefLabel` is a string *(it's literally `newtype` for a `String`)*, and it is your job to make sure whatever name you come up with wouldn't clash with other existing `RefLabel`s. So, well, it isn't much different from `getElementById()` you'd wanted to avoid.

Less compile-time safety for Halogen.

## Passing data from parent to child

You are probably unconvinced still, but this one is where things get really-really bad…

Suppose you want to pass something down the child. Like, imagine you're rendering a row in the table, and you want to pass the row identifier down to "edit the row" modal window.

In React it's trivial: you declare that the child expects a parameter `editMyRow :: Component RowState`, and you use it inside `React.do` monad. Oh, and pass the parameter from parent, that's it.

In Halogen you *(code examples are taken from the official tutorial)*:

1. Create a global id via type-level magic. Example: `_button = Proxy :: Proxy "button"`
2. Create special `Slots` type and make sure an id it accepts is same one as what you just created *(thankfully, at least you'll get a compile-time error on the mismatch)*. Example: `type Slots = ( button :: forall q. H.Slot q Void Unit )`.
3. Create `slot_` DOM element. Example: `HH.slot_ _button unit button { label: show count }`
4. You edit a bunch of places for the parent component to insert the new type *(which, if you separated all handlers to their own explicitly-typed functions for readability, will probably take  some minutes)*.
5. In the child you're proxying the input with `initialState`, and finally the input reaches the `render`.

Do you think it will work now, or did I forget something…? I mean, it compiles, and as you know in functional programming "if it compiles then it works"! Oh wait, it doesn't… Because you have to also add: 1. A receiving type and the code to handle it, and 2. initialize eval with something like `receive = Just <<< Receive`.

This last point is important: looking at my notes, apparently at some point I forgot to pass it, then things compiled but "for some reason" didn't work. Good luck finding why in the amount of abstractions you just had to build.

If that's not enough, looking at my notes there's another catch: passing input down the child can be done with `H.modify_` and `H.tell` — which one you chose? I hope it wasn't `H.modify_`, which tutorials may suggest, because now your component might randomly break. How? Well, it seems the state passed down this way may get cached to avoid calling `render`, and if you passed same data as the last time *(e.g. because a user invoked a modal window, then dismissed it, then tried invoking again)*, the child is no longer triggered. This isn't a problem in `H.tell`.

And by the way! Please read through examples: are you sure you understand every bit mentioned, every parameter, the `Proxy` thing, why we pass arguments the way we do…?  Because you have to maintain it later, you must to!

The amount of complexity compared to React is **insane**, I can't emphasize it enough! Type-safety added: zero, error-proneness increased, amount of time to just write this *(I'm not even mentioning maintenance)* to the point where it both compiles and works is just disproportional to the task.

## Meaningless initial states

This one is uniquely a Halogen problem. Suppose a component needs to render something as result of an effect. Imagine something trivial, like showing the current date.

In React there's a component creation phase, which happens before everything else and it allows you to execute such effect. So when component gets rendered, it has the correct initial state.

In Halogen you have to render a meaningless and excess "initial state", to be immediately overwritten once `handleAction` gets to run its `Initialize` phase. And often the "initial values" has no obvious initial state. Like, if you render a name, it doesn't make logical sense for it to be "empty", but you have to make it empty because of this peculiarity. And you have to edit this meaningless function every time you somehow change the component state type.

## Components refactoring

The complexity I just mentioned grows so much that when at some point you realize "my component could actually be broken up to two to reuse part of the code elsewhere", it may be easier and faster to just write new component from scratch than to try to decouple it from these humongous abstractions.

And this isn't just me, [look at this suggestion by one of Halogene's maintainers](https://discourse.purescript.org/t/best-way-to-handle-complex-state-in-halogen/1306/5?u=hi-angel) under a post where a user complains about complexity:

> It sounds like maybe you’ve broken things into components that don’t need to be? The sub-components that want to pass state back up to/receive state from their parent - do they have any additional state that would be awkward to manage in the parent? If not, I’d start collapsing back into the main component.

To paraphrase, it's basically "maybe you have too many components, try merging them". It's a situation which… well, maybe not "impossible", but at least one that's hard to get into with React, at least to the point where it becomes an issue. But even if it does become one, it goes without saying merging them back would be trivial in React but may easily be an ordeal in Halogen.

# Wasn't there a post "Halogen is better than React at everything"?

Yes, [there is such post](https://chrisdone.com/posts/halogen-is-better-than-react/), just wanted to mention. The title is a clickbait though, because this isn't what the post is about. The post is comparing "PureScript + Halogen vs TypeScript + React". IOW, it compares different programming languages rather than just frameworks.

# Conclusion

Halogen adds little to no type-safety over React Hooks *(the `ref` thing might be the only one but even that is disputable)*, error-prone in places, complicated to write and maintain. Lots of problems which just don't exist in React Hooks.

# P.S.

This may be one of my last PureScript posts. While my experiment with using PureScript did succeed, but at some point management decided they want cross-team commands, and the company has no PureScript projects. The decision didn’t involve any kind of technical discussion, it was sole top-management decision that they brought down.

During this work I contributed a lot, both in terms of basic questions and sometimes answers, as well as code-contributions to various core PureScript projects. In my spare time I also took maintainership/improved Emacs purescript-mode. Overall, I feel to have done good ecosystem improvements, and I hope this post will significantly simplify the life for newcomers too.
