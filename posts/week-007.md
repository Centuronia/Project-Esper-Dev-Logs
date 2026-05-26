---
title: First Looks
date: 2026-05-25
number: "007"
tags: gameplay, demo
excerpt: First gameplay preview of Project: Esper
---

Hello y'all,

Even though we haven't gotten any dev logs out in the past few weeks, we've nonetheless been hard at work on Project: Esper, and have an exciting announcement for today: our very first gameplay preview!

![Gameplay Preview - Complete](week-007/gameplay_complete.mp4)

Now of course this is just a very, very first look at the game and there's a lot still to be done. But a lot of stuff is going right behind the scenes to get this working, and there's a few features in particular we'd like to point out:

## Menuing

Project: Esper sports a layered, fully dynamic windowing system. As we show, the UI is divided into different menus and submenus. Each dynamically adapt and size based on their content, meaning wasted space is minimal. They also visually overlap each other so the window stack is clear. Lastly, windows support internal tabs so information can be easily organized. Of course, direct key-to-action bindings are also supported, and further customization options will be added later, as well as such necessary macros as move-to-point to keep things moving.

## Visibility

Keeping track of your surroundings is very important in any traditional roguelike, so we gave Project: Esper  a visibility and memory system to help with keeping stock of the situation. Tiles that are currently visible (from line of sight, scrying-type effects, or anything else) are displayed in full color, while tiles you've seen before but aren't right now are given a darkened appearance. This makes it very clear what is currently visible, and many traditional roguelikes have a kind of this feature. However, we also store the last glyph visible on any tile, and preserve and continue to display that. This way, if you previously saw a fiend off across the dungeon, you can remember it was over there to avoid it on subsequent trips through the same area. Or if you had to leave some loot behind, finding it again is faster than ever.

![Visibility Preview](week-007/visibility.mp4)

## Activated Effects

We've talked a lot about the crazy custom programming language driving the effect system of Project: Esper, and we can finally show off what that looks like! All the abilities used in this demo (and really throughout the whole game) were written in-language. The move abilities used to travel around are defined like:

```walk_north = active empty move player n 1 bimodal 5 5 ;```

The Boots of Maximum HP have the effects:

```hp_bonus = static empty stat_increase "max_hp" 5 ;
greater_hp_bonus = static empty stat_increase "max_hp" 15 ;```

Though the second ability is only granted while equipped. The Boots of Healing grant the heal_basic effect shown at the end, defined as:

```extern basic_heal = active proper_status
		player
		 snot hp_gt 75
	heal
		player
		/
			stat_of_proper
				"max_hp"
				player
			5
	 bimodal
		1
		1
;```

Which looks complicated but basically just has a condition of being below 75% health to use, and then for 1AP or RP heals the user for 20% of their health (which is a somewhat insane ability tbh, but was interesting to play with for the demo, especially since the BoMH were already effectively lowering the player's total health percentage by raising their max but not current health).

Even Equip and Pick Up are defined in the DSL (though they have to delegate to some custom logic due to their unusual functions not being something we want in the general randomization pool):

```extern pick_up = active empty
	with_single
		manual_single object_with_status
				entity_type "Item"
				sand 
                    snot held
					range_of_proper
						1 player
		"item"
		program "pick_up_lambda"
	action_only 5
;

extern equip = active empty
	 with_single
		manual_single
            sand
				snot equipped
				held_by_proper player
		"item"
		program "equip_lambda"
	action_only 5
;```

## Next Steps

All in all, this is a pretty exciting time! A lot of the work we've done over the past few months getting the custom game engine working is paying off, and getting from the state of having anything showing onscreen to this full demo only took a couple hours. So (with the exception of a few remaining major things that will take a while, *cough* NPC AI *cough*) development should move much, much quicker now. More importantly, now that the engine is compiling, we should be able to show off a lot more of what we do from post to post in these dev logs instead of just describing various abstract things.