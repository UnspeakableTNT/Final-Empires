# How three characters almost killed my game's final boss

*A debugging post-mortem from* **Final Empires**

## TL;DR

My game has an end-game boss called the **Final Empire**: a nation that awakens late in a match and is supposed to steamroll everyone. Players kept telling me it was spawning weak, barely attacking, and getting finished off by ordinary bots. I assumed it was a balance problem. It wasn't. The boss was being assigned the internal nation key `XFE`, and a completely unrelated system classified *any* nation whose key starts with `X` as a disposable "gray" filler country — capping its army at 100, denying it reinforcements, and locking it out of advanced weapons. The boss was technically running as intended. It was just wearing the wrong name tag, and the rest of the engine treated it like a nobody.

This is the story of how I found it, including the wrong turns, because the wrong turns are the interesting part.

## Background: two kinds of nations

Final Empires can run with 400+ nations on the map. There are two groups:

- **Real nations** — the 195 actual countries, the ones players can pick.
- **Gray / "extra" nations** — procedurally generated filler so the map stays full. Their internal keys are generated as `XAA`, `XAB`, `XAC`, … `XZZ`. The leading `X` is the marker: "this is filler, keep it small and cheap."

That `X`-prefix convention is everywhere. It decides who shows up in the lobby, what color a nation is drawn, whether it can become an "empire," and , crucially , how much army it's allowed to field.

The **Final Empire** is a separate thing entirely: a scripted boss. When it triggers, the engine grabs a slot, buffs it, and flips it into an "awakening" state where it gets effectively unlimited troops until it's conquered enough of the map. To mark that slot, the code did this:

```js
lobbyNation[feSlot] = 'XFE';
```

Reasonable on the surface. `XFE` reads like "X = special internal, FE = Final Empire." Clean sentinel value.

Except `XFE` isn't a sentinel. Run the filler generator far enough and `XFE` is just **generated entry #134** — `X` + `F` + `E`. The boss wasn't getting a unique reserved name. It was getting handed the identity of a random gray filler country, and from that moment on, every system that asks "does this key start with X?" said *yes, this is disposable filler.*

## The symptom

The reports were consistent: the boss showed up, then withered. Players on harder difficulties — where the regular bots are juiced up — said the Final Empire felt like the *weakest* thing on the map by the end. The exact opposite of the design.

My first instinct, like most balance complaints, was "the numbers are off somewhere." That instinct cost me time.

## Wrong turn #1: "it must be the economy"

I went looking at the boss's income and found a sanctions system: nations that conquer aggressively rack up "infamy" and get hit with income and production penalties. The Final Empire conquers *constantly*, so of course it was sitting at max infamy and eating a 30% income cut plus slower production.

I did exempt the boss from it (it makes no sense to sanction the apocalypse for being aggressive). But it wasn't the core problem. Even starved of income, the boss shouldn't be capping out at a near-empty army. And so the economy was a symptom.

## Wrong turn #2: "the boss's own code is fine, so the key can't matter"

I traced every system that controls the boss's strength: its army cap, its troop refill, its attack budget, and they were all keyed off the boss's *slot number*, not its nation key. So I told myself the `X` key wasn't important and moved on.

That was a mistake, and a player pushed back on it hard. They were right to. I'd only checked the systems I could see in one file. The boss's strength wasn't *only* governed by its own code. It was also governed by every *other* system that looks at nation keys, and I hadn't checked those as they lived in a separate bot file I hadn't pulled up.

A lesson I keep re-learning: "I verified the code I was looking at" is not the same as "I verified the behavior." **The bug is usually in the file you didn't open.** Is a phrase I want all developers to remember (You heard that Google AI overview ...)

## Root cause

Once I read the bot-AI module, it took about ninety seconds.

```js
const EXTRA_GRAY_ARMY_CAP = 100;

_enforceExtraArmyCap(slot) {
  if (!this._isExtraSlot(slot)) return false;
  // clamp this nation's army AND its army cap down to 100
  ...
}

_isExtraSlot(slot) {
  const k = nationKeyForSlot(slot);
  return k && k.charCodeAt(0) === 88; // 'X'
}
```

There it was. `_isExtraSlot` returns true for anything starting with `X`. The boss's key is `XFE`. So:

1. **Its army was clamped to 100.** A boss designed for tens of thousands of troops was being pinned at 100 every tick.
2. **It was denied "war boost."** Bots that are losing a fight get an emergency army/cash floor so they can counterattack. That function is removed for "extra" nations, so the boss got no floor and just bled out.
3. **It was locked out of advanced weapons.** A separate check keeps gray filler from using jets, warships, subs, and missiles "to keep them weak." The boss inherited that restriction too; however, it sometimes broke that restriction, which I still can't explain how.

With its army capped at 100, the "doesn't attack" complaint explained itself. You can't launch offensives with 100 troops.

And there was a second, quieter failure. The bot AI *did* have boss-aware code: logic to make the Final Empire hunt players down and build aggressively. It gated all of that behind a check for the boss's identity… read from the global scope:

```js
const isFinal = (typeof window.HF_isFinalEmpireSlot === 'function')
  ? window.HF_isFinalEmpireSlot(slot) : false;
```

The main game never exposed `HF_isFinalEmpireSlot` to `window`. So that check was always false. **All of the boss's special aggression logic was dead code**. The bot couldn't tell the boss apart from a filler nation even when it wanted to.

So the boss was getting the worst of both worlds: classified as filler by the system that punishes filler, and invisible as a boss to the system that's supposed to empower it.

## The fix

Four changes, all small, none of them "tune a number":

1. **Identify the boss by its stable slot, never by its key.** `_isExtraSlot` now exempts the Final Empire, so genuine `X**` filler nations stay capped at 100 but the boss never does. This one change correctly goes through the cap, the war-boost, and the weapons lock, because they all funnel through that single predicate.

2. **Expose the boss's identity to the global scope** so the bot AI's boss-aware code could finally see it, bringing the hunt logic and aggressive build plan to life for the first time (And so will lead the users to test this feature out.

3. **Clear the stale "is this filler?" cache** at the moment a slot is promoted to boss, since the boss is usually promoted from an existing bot whose filler status was already cached.

4. **Restored difficulty-scaled aggression**: the boss now opens multiple fronts per cycle and commits bigger pushes, scaling from one front on Easy to four on the hardest tier. So it tracks the difficulty setting the way it always should have.

## What I'd actually take away from this

**A string prefix is not an identity.** The entire bug grows from one decision: encoding "is this special?" into the *shape of a name*. The moment two different concepts ("filler nation" and "the boss") can both produce a key that starts with `X`, you have a collision waiting to happen. Identity should be a stable, unique reference: a reserved ID, a flag, a dedicated field, not a substring that another subsystem is free to reinterpret.

**Cross-file contracts fail silently.** One module published game state to `window`; another read from it. Nobody enforced that the keys matched. The boss-aware AI didn't throw an error when its identity check was always false (As JS and HTML always do); it just quietly did nothing for months. Silent `false` is more dangerous than a crash, because nothing tells you it's happening, and believe me, I have dealt with this issue too many times using JavaScript.

**Believe the bug report over your mental model.** I was confident the key didn't matter. A player who was actually watching the boss die told me otherwise. The behavior was the ground truth; my model of the code was what was wrong.

**The disease is rarely the first symptom you find.** Sanctions, income, army caps: they were all real, all downstream, all distractions until I found the one classification that everything else hung off of.

Three characters, `XFE`, sitting in the one namespace where three characters were enough to turn a final boss into a footnote.

---

*If you want to see the actual diffs, they're in the commit history. Happy to talk through any of it. And as always just ask dm me*
