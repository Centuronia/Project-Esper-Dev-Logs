---
title: Effect Engine
date: 2026-03-27
number: "002"
tags: technical, engine
excerpt: Technical details on the Effect DSL driving Project: Esper
---

Hello y'all,

As referenced in last week's post, Project: Esper is unique in that it runs entirely on a custom Effect DSL (domain-specific language). Today I thought I'd go more in-depth on what that looks like technically, and then maybe next week we'll talk a bit more about the impact that has on the game.

## Overview

As the name "Domain-Specific Language" implies, a DSL is essentially a programming language, but instead of being maximally expressive and Turing-complete like normal languages, it is designed to be very good at a very specific thing. The goal is to create a grammar (set of words and the rules that connect them) that is as simple and easy to write in as possible while fully representing whatever it is meant to. When written well, this makes doing *stuff* in the language very, very simple, at the cost of having to set it up once. It also has the nice property that, since the rules are already written out, anything that can be expressed as valid code in the DSL will translate into some valid result in the interpreter. This is a very useful property, as we'll see later.

## Philosophy

As the lead developer of Project: Esper, my personal software philosophy is all about abstraction. Abstraction is king. Essentially, a good program is like a wall. You've got a bunch of different blocks, and then you connect them all together in various ways. But then you dive deeper and see that each block is really a smaller copy of the wall, with its own blocks and connections. So the goal, when programming good applications, is to get all those basic, super-small blocks working independently on their own and representing whatever they ought to very well with very clear and simple ways to stitch them together (this is commonly referred to as the API). And the nice thing about writing code this way is A. that any given *thing* is in exactly one place, so whenever something needs to be changed, added, optimized, etc. it is very obvious where to go and all that needs to be changed, and B. because the API defines a contract, I can write each block independently, trusting that when each other block gets written it will obey the contract it's meant to, and thus the required shape of each block is very simple and clear as well. The reason I bring this up is because a DSL is this principle taken somewhat to an extreme. It allows you to tokenize these blocks and invoke them without writing any code. For instance, given the requisite DSL I could say ```move player 5``` and trust that the "move", "player", and "5" atoms will all be represented correctly such that this line properly invokes a movement action on the player with value 5. But the beauty is that this enables a combinatoric explosion. Because I could swap out any of those 3 atoms for any number of different possibilities, and trust that because each atom obeys its own contract, everything will Just Work™, making writing anything expressible in the DSL, as aforementioned, extremely simple.

## Effect DSL

As I've mentioned, the entire "effect" system for Project: Esper is built on a custom DSL. Essentially, there are 3 kinds of effects:
- Static effects apply constantly, and modify various numeric stats on an entity whenever they are accessed.
- Triggered effects apply some output whenever an event condition is met.
- Activated effects apply outputs the same way as triggered effects, but they are manually activated as actions or reactions by characters instead of happening automatically.
All three kinds of effects can be innate to an entity, applied by other effects or zones, or added by equipped items or items in inventory. Additionally, all three kinds have associated "predicates" that define conditions under which the effect will be active at all. For example, consider the effect:
```while you are on fire, gain the activated ability for 1 action point: heal 5hp```
Here, "while you are on fire" is the predicate. Predicates can be empty, but their existence is vital to the premise of the game. Because all (non-trivial) effects are predicated, no effect can (trivially) reduce attack surface. Look at that example. The character has a very nice regeneration ability they can activate, but it's contingent on being on fire. So if the player wants to heal, they have to figure out how to be on fire, gain fire resistance, etc. Conversely, if their enemy is disparaged by their constant healing, they can attack the fire by dousing them, removing the fire source, etc. The attack surface changes from a broader one (attack hp) to a more specific one (attack the "on fire" status).

Returning to the DSL, an effect is assembled in a tree structure (the technical term is an Abstract Syntax Tree). This is parsed from the raw DSL code by paying attention to the language's grammar and what tokens must belong to what class (for anyone interested, this is essentially the process ***every*** programming language goes through during compilation, and the reason that things like code error detection, compiler warnings, and writing in plaintext and any language other than Assembly can be things). The cool thing is that the AST is internally completely defined, so we can guarantee that any code parsed into the tree must be valid and will run in-game.

## Examples

I'm going to spare writing out the entire vocabulary for the DSL, since at this moment there are 208 valid tokens and counting, but I'll still try to show what it looks like to write in. We can write something like:
```
extern fire_boost =
    static
        proper_condition player "burning"
        stat_increase "fire_resistance" stat_of_proper "STR" player;
```
And that will, in 121 characters, define an effect that, while burning, boosts the affected creature's fire resistance by an amount equal to its strength score. This gets translated into an AST like:
```
fire_boost: 

(Top) static
  (Predicate) proper_condition
    (RelativePO) true_proper
      (ProperO) player
    (Condition) condition
      (Data) burning
  (Static) stat_increase
    (Stat) stat
      (Data) fire_resistance
    (Constant) stat_of_proper
      (Stat) stat
        (Data) STR
      (RelativePO) true_proper
        (ProperO) player
```
You'll notice it fill in the missing connections between atoms where needed, as well as perfectly calculate the inheritence tree. The gameplay then becomes very simple: whenever we query "fire_resistance" we just iterate over all the objects' static effects, see that the player is burning, and then apply the stat_increase denoted by the effect. Everything is perfectly modular and simple.

## Procedural Generation

The not-so-subtle reason we care so much about guarantees of validity is that if the DSL has clear rules about what will produce valid effects, and any such construction will be valid, we can randomly generate effects, and do so very trivially. Essentially we can literally just roll on tables for each atom type in the language until we reach an end, and *presto* we have a valid effect (in practice there will be significant weighting on these tables to bias against more convoluted and rarer atoms, but the principle is the same). For an example:
```
1. <Top level>
2. If <Predicate> Then <Static Effect>
3. If <Proper Object> has <Status> Then <Static Effect>
4. If the entity that caused the most recent <Event> has <Status> Then <Static Effect>
5. If the entity that most recently set something on fire has <Status> Then <Static Effect>
6. If the entity that most recently set something on fire is poisoned Then <Static Effect>
7. If the entity that most recently set something on fire is poisoned Then decrease <Stat> by <Constant>
8. If the entity that most recently set something on fire is poisoned Then decrease visibility by <Constant>
9. If the entity that most recently set something on fire is poisoned Then decrease visibility by the minimum of <Constant> and <Constant>
10. If the entity that most recently set something on fire is poisoned Then decrease visibility by the minimum of the number of visible <Object> and <Constant>
11. If the entity that most recently set something on fire is poisoned Then decrease visibility by the minimum of the number of visible goblins and <Constant>
12. If the entity that most recently set something on fire is poisoned Then decrease visibility by the minimum of the number of visible goblins and 2
```

As this demonstrates, we end up with a fairly interesting effect:
"If the entity that most recently set something on fire is poisoned Then decrease visibility by the minimum of the number of visible goblins and 2"
This seems to be an effect you'd want to inflict on your foes when they stand near goblins to stealth around them, and you probably to save time activate it by setting something on fire and then poisoning yourself (though if you could get something else into that state, that could work too). Is it a useful ability? Probably not. But is it *interesting*? Certainly. Does it get the player's wheels turning about how it could be useful and manipulated? You betcha! The mark of a good roguelike is when the player is tasked with taking a bunch of random effects and mechanics and producing a method out of the madness. Simultaneously, since we can generate random effects, we can piggyback off of that to generate random items and enemies as well, in a way no other game has really done.