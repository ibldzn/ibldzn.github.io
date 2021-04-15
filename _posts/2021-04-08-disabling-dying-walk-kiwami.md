---
layout: post
title: "Disabling 'Dying Walk' in Yakuza Kiwami"
date: 2021-04-08 09:15:21 +0700
categories: re
author: ibldzn
tags: [yakuza kiwami]
---

![Kiryu Dying](https://i.imgur.com/keRam7j.png){: .center}
_Kiryu having stomachache_

# Introduction

---

I recently installed Yakuza Kiwami again just because I miss ~~ab~~using [Tiger Drop](https://yakuza.fandom.com/wiki/Komaki_Tiger_Drop).

They nerfed this move in the next series released after Yakuza Kiwami because how overpowered it was,
not to mention you can literally [one hit](https://youtu.be/rFlmc8Mr8Qg) bosses with 4 health bar if you buff yourself using [Time For Destruction](https://yakuza-archive.fandom.com/wiki/Time_for_Destruction),
equipping [Payback Ring](https://i.imgur.com/dxRrTZn.png) and [Dojima Family Amulet](https://i.imgur.com/9WKnCLa.png) while having low health.

Because I love ~~ab~~using this move so much, I prefer walking around town with low health to instantly finish [Majima Everywhere](https://yakuza.fandom.com/wiki/Majima_Everywhere) fight,
but the problem is once your health is so low to the point the health bar is flashing
Kiryu will do this ["Dying Walk"](https://i.imgur.com/857Nkga.gif) which completely slow down
your movement speed and not allowing you to run at all.

# Into the fun part

---

To find out how the game decide when Kiryu should do that "Dying Walk" we first
need to find where's our health laid in memory, the game has to access it somewhere
if Kiryu only does that walk everytime his health is flashing.

I've actually reverse engineered the game a little bit a few months ago to make this [simple mod](https://github.com/ibldzn/yakuza-kiwami-heat) that fills your heat bar after pressing R3 and completely deplete your heat after pressing L3. Having this [information](https://github.com/ibldzn/yakuza-kiwami-heat/blob/1f1bf3d4ce5d28f3052c570718edd8e9c31b322b/yakuza-kiwami-heat/src/hooks/hooks.cpp#L7) with me, it's easy for me to locate that function, again. Also, [this](https://www.unknowncheats.me/forum/general-programming-and-reversing/171994-understanding-pattern-scanning-concept.html) is a great writeup if you want to understand how pattern scanning works.

I found the so called "heat_manager" function which gets called every tick when you're in a fight.
Here's the dissasembled code

![Dissasembled Code](https://i.imgur.com/XjINQL6.png)

`player` is a pointer to a struct holding all of player's data, by adding `0x1C`
to the `player` pointer and reading the value as a float we get our current heat value,
that being the case it is more than likely our health is somewhere in that struct.

# Hunting where our health is in memory

---

Yakuza Kiwami doesn't have [ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization) enabled,
thus making it easier for me to move from decompiler to debugger without having to rebase anything.

Decompiler said the `*(float *)(player + 0x1C) = amount;` line we see above is at `0x000000014047A272`

![Address](https://i.imgur.com/pCNKxiY.png)

Our `player` pointer is stored inside `rax` register, we should get whatever
is in the `rax` register when this line gets executed,
let's put a breakpoint at the address I've mentioned above and get into a fight in-game.

![Breakpoint Hit](https://i.imgur.com/LxhFjvX.png)
_We hit the breakpoint_

`0x00007FF4D0AF2970` inside the `rax` register is pointing to our `player` struct,
let's open up the `Dissect data/structure` menu in Cheat Engine
and put the value we just got into the address tab, aanndd..

## We got our player struct

![Player Struct](https://i.imgur.com/n9vLdKS.png)

There are bunches of 0 and unknown value there,
but we know for sure that `0x1C` is where our heat value is stored, let's label it.

![Heat Label](https://i.imgur.com/03mtpfz.png)

## We found our [possible] health value

After getting hit by enemy in-game and drinking "health potion" the value at
offset `0x14` changes, this is very likely to be our health!

Let's add it to our address list

![Adding Health Address to Address List](https://i.imgur.com/FDqNgAd.png)

![Health Address in Address List](https://i.imgur.com/roTGHYy.png)
_Such a weird value for a health, huh?_

# Let's find out what accesses our health value

---

Now that we have our health location in memory, we can right click on that address
and click on `Find out what accesses this address`

![Find out what accesses this address](https://i.imgur.com/ThwrRoF.png)

Going back in game for a few seconds then I immediately saw this

![What accesses this address](https://i.imgur.com/6oTF0hZ.png)

Let's gamble our luck with the first one.

Going to address `0x140319A16` in dissassembler points me to this line

![0x140319A16](https://i.imgur.com/B14CuV6.png)
_Hex Mismatch, What???_

This function looks interesting, it divides
our current `health * 100` with something,
let's try spoofing the return value stored in the `xmm0` register

![xmm0 register](https://i.imgur.com/5GOhmPb.png)

![Spoofing Success](https://i.imgur.com/7wK1Sbo.gif)

Oh boy we're lucky we found this in our first try,
if we set the value of the `xmm0` register to be `xmm1` (the divider),
we no longer see the flashing health and we're able to move normally in game even in low health.

Let's refer to that function as `health_something` (I don't have a good naming sense,
nor do I know where the game gets the divider value, I'm sorry).

# Side effect

---

Returning big value from `health_something` function causes some side effect,
We can't do any heat actions which are only do-able when our health is flashing
[(like this one for example)](https://www.youtube.com/watch?v=D7ov8-uPY6E).

Uh, oh.. What should we do??

Well, the return value from `health_something` must be used from multiple places,

![xref](https://i.imgur.com/yw0npLm.png)

and we're right, finding the cross reference in decompiler shows that function
gets called from quite a few places.

One of those functions must've decide whether Kiryu should do the "Dying Walk",
but which one?

Well, time to fire up [x64dbg](https://x64dbg.com/)
so we can log the [return address](https://en.wikipedia.org/wiki/Return_statement)
without needing to resume program execution after breakpoint hit.
(not sure if Cheat Engine debugger has this feature).

## Narrowing down the call

Let's go to `sub_140319A00` (`health_something`) in x64dbg,
(`sub_` is a prefix for `subroutine` a.k.a `function`,
those numbers after the `sub_` prefix is the function address,
in this case we should go to address `140319A00`).

Let's put a conditional breakpoint in that address

![Conditional Breakpoint](https://i.imgur.com/Qh0tbD4.png)

As we can see in this [Stackoverflow answer](https://stackoverflow.com/a/62185229)
the function's return address is pointed by the `RSP` register,
that means in order to get the actual return address we need to dereference the value
at `RSP` register. Let's log `[RSP]` value by typing this in the conditional breakpoint setting

![Conditonal Breakpoint Setting](https://i.imgur.com/qI3uLKq.png)

- `Break Condition: 0` (We just need to log the value at `[RSP]` and we don't want
  to go back and forth to x64dbg just to resume execution)
- `Log Text: return address: {p:[RSP]}` (A log text with [x64dbg string formatting](https://help.x64dbg.com/en/latest/introduction/Formatting.html))
- `Hit Count: 1264` (It's just how many times this line gets executed since we placed the breakpoint)

Going to the `Log` tab in x64dbg, we immediately see a lot of logged value,
but it seems like there's only 2 unique value there

```
...
return address: 0000000140480759
return address: 0000000140480759
return address: 0000000140480759
return address: 00000001404F54E6
return address: 0000000140480759
return address: 0000000140480759
return address: 0000000140480759
return address: 00000001404F54E6
return address: 0000000140480759
return address: 0000000140480759
return address: 0000000140480759
return address: 00000001404F54E6
...
```

Well, let's first see what's inside `0000000140480759` in dissassembler.

![0000000140480759](https://i.imgur.com/TcmYVIm.png)

It's comparing a global variable with the returned value from the `health_something`
subroutine, could this be the one?! Here's how the function looks like in asm

![0000000140480759 asm](https://i.imgur.com/9GwExka.png)

The line that caught our interest is [`setae al`](https://c9x.me/x86/html/file_module_x86_id_288.html)
which is essentially the return value.

Let's change it to:

- `mov al, 1` (`return true;`), and
- `mov al, 0` (`return false;`)

and see what happens in game.

![return true](https://i.imgur.com/MbXbzS5.gif)

Apparently if we return true in that function we'll see our health bar flashing,
but it doesn't affect the "Dying Walk" at all, and to my surprise you can do any heat actions
that requires you to have flashing health bar even though your health is full! We could exploit this by [hooking](https://en.wikipedia.org/wiki/Hooking)
that function and returning `true` or `false` whenever we want! Great stuff.

Here I tried returning false while having low health but the "Dying Walk" is still there,
even though my health bar is no longer flashing.

![return false](https://i.imgur.com/KxnVMdB.gif)

Now that we know what it does, let's name this function as `should_flash_health_bar`

![should_flash_health_bar](https://i.imgur.com/36Pb0oj.png)

Well, that wasn't the exact function we're looking for but we're glad we found it.
Let's check the other address at `00000001404F54E6`

![00000001404F54E6](https://i.imgur.com/NcVdCEO.png)

It's comparing against the same global variable used in `should_flash_health_bar`!
Well, I guess we could safely assume that `dword_140FCE630` is a some
sort of thresholder value.

## Spoofing the function return value

Again, the function returned a boolean, we could try the same method we used with
`should_flash_health_bar`, spoofing the return value and see what changes in game..

Also, here's how it looks in asm, what caught our interest this time is the
[`setbe al`](https://c9x.me/x86/html/file_module_x86_id_288.html) line.

![00000001404F54E6 asm](https://i.imgur.com/5sjGuF4.png)

Spoofing the return value to be false and theenn...

### ["DYING WALK" IS GONE!](https://imgur.com/Slqakdx)

This function's return value is used to determine if Kiryu should do the "Dying Walk".
Let's rename `sub_1404F54A0` to `should_dying_walk`.

![should_dying_walk](https://i.imgur.com/H3dEvkY.png)

# Finishing Up

---

Now that we got what we wanted all that's left is to make the game execute what we want,
we could just dump the process with the bytepatch we just did,
but that's such a "hacky" way to do it especially for game hacking,
let's make a DLL that will hook `should_dying_walk` and returning false everytime.

I'm using [minhook](https://github.com/TsudaKageyu/minhook) library for this

```cpp
#include <Windows.h>
#include "include/MinHook.h"

// function prototype
using fn = bool(__fastcall*)(__int64);

// pointer to the original should_dying_walk
static fn o_should_dying_walk = nullptr;

bool __fastcall hk_should_dying_walk(__int64 a1)
{
  // call the original first otherwise it won't work.
  o_should_dying_walk(a1);
  return false;
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD dwReason, LPVOID lpReserved)
{
  if (dwReason == DLL_PROCESS_ATTACH)
  {
    DisableThreadLibraryCalls(hModule);
    MH_Initialize();
    MH_CreateHook(
      // 0x1404F54A0 is the function address for should_dying_walk function.
      reinterpret_cast<LPVOID>(0x1404F54A0),
      &hk_should_dying_walk,
      reinterpet_cast<LPVOID*>(&o_should_dying_walk)
    );
    MH_EnableHook(MH_ALL_HOOKS);
  }
  else if (dwReason == DLL_PROCESS_DETACH)
  {
    MH_Uninitialize();
  }
  return TRUE;
}
```

That is a really basic DLL with no error handling whatsoever and hardcoded offset (they never change anyway, but yea..)

Well, I guess that's it. We were looking for copper but also found gold along the way.
Thank you for reading my first article!
