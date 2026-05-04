---
title: Target Selection
date: 2026-05-03
number: "006"
tags: technical, gameplay
excerpt: NPC target selection and the emergence of personality and nonpredictable interactions
---

Hello y'all,

Continuing from last week's post about reflection and how we set up the NPC AI to choose which actions to take, today we're going to take a moment to talk about the other half of NPC AI: target selection of those chosen actions. This is pertinent because we've reached the point where it's time to implement, and we have a very interesting system cooked up.

## Selection Types

First off, the only time that any of this "selection" happens is when a specific creature gets/needs to select parameters for a specific action or reaction it's planning to take. There are four selection functions that a given action might call:
1. select_one, where a list of entities is provided and the entity must choose one. For the player, they are presented with a list of entities they can move through and look at before selecting one.
2. select_direction, where the entity selects one of the 8 cardinal or intercardinal directions. For the player, they just press the corresponding direction key (which will be bound to the same key as moving in that direction).
3. select_tile, where the entity chooses any tile on the map. The player is given a cursor they can move around and look at stuff with before selecting the current position.
4. select_visible_tile, which is the same as select_tile except only tiles the entity/player can currently see are valid selections.
As of now, this small quartet constitutes the entire breadth of possible selections in the game.

## Disposition

The first layer of the NPC selection algorithm is compilation. Once a given DSL effect is loaded into an AST during the intial game launch, we also iterate through to find all the output effect atoms. As a rule, these selection nodes can only be descendants of output effects in activated effects, since they don't really make sense for the more continuously-evaluated predicates or trigger trees, and manually targeted triggered abilities could get very annoying very quickly ("Every time any creature moves I have to manually say 'no, I do not want to fireball myself'"...) and represent intention in a way that triggered abilities don't really have. Thus, we know that every selection will be directly related to a specific effect atom or set thereof.

These allow us to determine the general type of *thing* we're doing with the given selection, and most importantly, whether it is likely an offensive or defensive selection. For example, "attack" targets one entity and that entity should probably be one we don't like, while "heal" targets one entity and it probably should be one we do like, while "teleport" could go either way. We define this as the **disposition** of the effect atom and thus its child targets and selection nodes. Note that sometimes the disposition of two of an atom's targets will be different (Attack actually has two targets: one for who does the attack and one for who receives it, and these have different dispositions).

## Base Point

One way of looking at all four selection types is that they can all be defined by the location of a single entity. For select_one, literally just choose that entity. For select_direction, the direction is the direction from the selecting entity to the positional entity. Select_tile and select_visible tile are both just the position of the entity. Looking at the four modes in this way also has the advantage of fitting nicely with a lot of information we already have available, as the engine is very entity-centric (as entities are every object in the game).

So we can essentially ignore the specific selection type and follow the same remainder of this algorithm in all 4 cases, just understanding that at the end we will get our final selection via the respective process outlined above.

First we need an initial set of entities. For select_one, there's the conveniently passed-in set of entities we're given. For the others, we just take the set of all visible entities (as NPCs oughtn't be aware for these purposes of entities they can't see). Then we filter out this initial set by disposition; if we're trying to arrive at something related to friendly entities, only keep those ones, and likewise for hostiles. Then all that's left is to arbitrarily pick one of the remaining entities. Theoretically any choice is rational, as the NPC shouldn't end up casting Power Word: Kill on any allies since they aren't hostile. Nevertheless, we want to be a little more systematic in how we make this selection.

## Personality

We define 17 different selection modes, which are ways of filtering or selecting from the remaining entity pool. Each is associated with a particular intelligence threshold, which is essentially the representation of the tactical load required to effectively follow this mode. It also means that more intelligent NPCs will generally use more interesting targeting mechanisms (it does not, however, mean that they will be chosen more intelligently). Still, each NPC is assigned a particular selection mode per selection node in each effect, from their available choices, that they will always use for that particular selection node. This is a weighted table, so less complex or idiosyncratic modes will be more common. The process essentially represents the NPCs particular personality and how they have learned to apply their skills. Suppose there are three mind flayers with the ability "Teleport target entity to target point". The first might have learned to use this ability to teleport itself towards enemies to attack. The second teleports its enemies towards its allies instead. And the third simply reserves the ability for beating a speedy retreat and guerilla warfare. The same ability can be used tactically in different ways, and the existence of diverse targeting modes enables the various NPCs the player encounters to show real personality, make the most of their abilities, and act almost adversarially. The full table is below:

| Mode | INT Threshold | Description |
| ---- | ------------- | ----------- |
| Random | 0 | Chosen randomly from entity set |
| Closest | 0 | The closest entity in the set |
| Self | 0 | Targets itself if possible |
| Most Hated | 2 | Targets least liked entity |
| Least Hated | 3 | Targets most liked entity |
| Last Damaged Me | 4 | The entity that most recently damaged it, if applicable |
| Weakest | 6 | The weakest entity (based on tier/level/equipment) |
| Farthest | 8 | The farthest entity |
| Last Buffed Me | 9 | The entity that most recently applied a positive status effect or healed it, if applicable |
| Cluster | 10 | The center point of all entity positions in the set. In the case of select_one, then finds the closest entity to the center |
| Least Damaged | 11 | The least damaged entity in the set |
| Most Damaged | 12 | The most damaged entity in the set |
| Strongest | 12 | The strongest entity (based on tier/level/equipment) |
| Damaged Me Most | 13 | The entity that has done the most total damage to it of the ones in the set, if any |
| First Damaged Me | 14 | The first entity that damaged it, if applicable |
| Exposed | 14 | The entity in the set farthest from the center point of all entity positions in the set |
| Interruptable | 15 | Any entity that is currently activating an ability but has not finished yet |

In case a mode isn't applicable or can't find a valid target in the set (most often due to dispositional differences), it will fall back to the entry directly above it. Before reaching Random in this way (but not if Random was selected as the entity's behavior for this selection node), the selection will attempt to be cancelled.

## Inverses

For selection types except select_one, each selection mode can also be used in its normal or inverse modes. Currently the plan is for inverse mode to be selected randomly 25% of the time, for any selection mode available except the highest-ranked. This means that Interruptable cannot be inverted (subject to change). An inverse selection is applied after the initial base entity or point is decided. Then:
1. In the case of select_direction, the exact opposite direction is used, which is mathematically the direction from the selected entity towards the selecting one
2. In the case of select_tile, the tile mirrored on the opposite side of the selecting entity from the selected one is used
3. Select_visible works the same as select_tile, but chooses the closest visible tile to the new selected one
These all demonstrate the same idea, but are important for things like running away.

## Conclusion

The goal is for NPC encounters to be both interesting and rational. By encoding specific behavior patterns per-NPC in this way, we can hopefully provide both consistency and creativity. You can trust that NPCs will use their abilities in reasonable and advantageous ways, but how exactly they will choose to do so is a product of their own thought processes and personality. You might just encounter a Master Lich that really, really likes to fireball sheep.