---
title: 'Your first Unity IL2CPP hack'
description: 'The basics for hacking Unity IL2CPP games'
pubDate: 'Mar 28 2024'
heroImage: '/gumballrun.jpg'
---

## What you can expect to learn:
- Dumping IL2CPP unity games using perfare's IL2CPPDumper
- Finding offsets
- Hooking functions
- A little bit of DNSpy
- How to use [Minhook](https://github.com/TsudaKageyu/minhook)

---

## Some things you will need are well, for starters:
- A functioning frontal lobe
- Basic C++ knowledge
- DNSpy
- An IDE which supports C++
- IL2CPPDumper

## (Not a lot is it now?)

---

The first step in hacking a game is knowing which game you want to hack.

For example, I am going to create a hack for a game called [gum ball run](https://store.steampowered.com/app/2124930/Gum_Ball_Run/), where the point of the game is to beat your enemies by being the first to get to the finish line.

---

For a game such as this speed is an important factor, so let's change the speed value!

Let's dump the game using [IL2CPPDumper](https://github.com/Perfare/Il2CppDumper/releases/download/v6.7.40/Il2CppDumper-net6-v6.7.40.zip):

First run <img src="/IL2CPPDumper.exe.png" alt="IL2CPPDumper.exe"> file afterwards select <img src="/GameAssembly.dll.png" alt="GameAssembly.dll"> file then lastly select <img src="/global-metadata.dat.png" alt="global-metadata.dat"> file

<p align="center"><u><i>you can find GameAssembly.dll in your games root directory
and global-metadata.dat in GameName_Data\il2cpp_data\Metadata</i></u></p>

If there were no errors, there should be a bunch of new files in the dumper's

---

The main file we are looking for should be inside of

<p align="center"><img src="/Assembly-CSharp.dll.png" alt="Assembly-CSharp.dll file"></p>

Once you have found it, drag it into DNSpy like so

<p align="center"><img src="/WhereToDragAssembly.png" alt="WhereToDragAssembly.png instruction"></p>

You will be met with a whole lot of things, don't let this intimidate you tho! These are ***(mostly)*** harmless classes which ***(hopefully)*** won't bite <img src="/Dave.png" alt="Dave" width="30px" height="30px">.


Using the search function we can find things like functions and types, in our case we will search for a speed type/field, usually it would be under a class which controls the player, e.g something like ***PlayerController*** or ***CharacterController***.

<p align="center">In the case of Gum Ball Run it is PlayerCharacter:</p>

<p align="center"><img src="/PlayerCharacterClassInDNSpy.png" alt="PlayerCharacterClassInDNSpy.png class"></p>

---

A lot of unity games are obfuscated to prevent people from reversing the game and making cheats/hacks for the game.
Unfortunately Gum Ball Run is obfuscated, but that won't stop me!


<p align="center">Example of obfuscated games usually contain a lot of dummy functions:</p>

<p align="center"><img src="/ObfuscatedGameExampleFunctions.png" alt="ObfuscatedGameExampleFunctions.png example"></p>

---

<p align="center">PlayerCharacter contains two fields/types we will be changing for now:</p>
<p align="center"><img src="/WalkAndRunSpeedFields.png" alt="WalkAndRunSpeedFields.png fields/types"></p>

Usually these fields/types are used inside of an update function. In the next article I will be covering how to decompile these function to see what they actually do and e.g see if they actually update the walk/run speed in there

<p align="center"><u><i>(Often times the update function does check and update the speed of the player.)</i></u></p>

Now that we know which function updates the player's speed and what the fields for the walking speed and running speed are, we will hook the function and change those fields.

<p align="center"><u><i>You can also view il2cpp.h for the structs and fields incase you really can't find anything in DNSpy!</i></u></p>

---

Clone [this](https://github.com/rdbo/ImGui-DirectX-11-Kiero-Hook) repository to get started, it contains ImGui and Minhook.

<p align="center"><u><i>We are using ImGui as a gui framework for our hack and
Minhook for hooking and replacing functions with whatever we want.</i></u></p>

<p align="center">Once you are in visual studio, press this button to make it easier to navigate your project:</p>

<p align="center"><img src="/NiceThingToEnableToImproveProjectNavigatability.png" alt="NiceThingToEnableToImproveProjectNavigatability.png button"></p>

<p align="center">Navigate to main.cpp and scroll down until you are at this part:</p>

<p align="center"><img src="/GuiCodeLocationPicture.png" alt="GuiCodeLocationPicture.png image"></p>

<p align="center">This is where you will create/modify your gui.</p>

<p align="center"><u><i>As mentioned at the top of this page, you are expected to have a small amount of knowledge on C++, I won't explicitly talk about basic C++.</i></u></p>

---

Seeing as we are going to change fields from the game we will need to include the il2cpp.h file which was included in the dump.

***FYI***. This is a massive multi-million line file so I do not recommend opening it inside of visual studio, instead use sublime text 3 or visual studio code.

---

<p align="center">We are now going to create the hook, let's put it inside of </br><b><i>modules->misc->speedhack.h</b></i></p>

First we are going to recreate the original function:

```cpp
typedef void(__fastcall* JoinMatch_Orig)(void* __this, void* method);
JoinMatch_Orig JoinMatch_Original = nullptr;
```

We will then make the detour function, the function which will replace the original.

```cpp
void __fastcall playerCUpdateDetour(PlayerCharacter_o* __this, void* method) {
    // Set the walkSpeed field
    __this->fields.walkSpeed = globals::Speedhack;
    // Call the original function so it actually updates.
    JoinMatch_Original(__this, method);
}
```

<p align="center"><u><i>(We set the walkSpeed field to our global which contains a float with our desired speed, then call the original function.)</i></u></p>

Before hooking the update function we need to know the offset to it, we can get this in DNSpy like so:

<p align="center"><img src="/UpdateFunction.png" alt="UpdateFunction.png offset"></p>
The RVA (Relative Virtual Address) is the offset we are looking for.

---

Alternatively we can get it by opening the script.json file included in our dump and search for the function.

<p align="center"><img src="/TheFunctionInscript.json.png" alt="TheFunctionInscript.json"></p>

```json
"Address": 7624736
```

This is the decimal value of the address, to get the RVA we need to convert this to hexadecimal and put 0x before it e.g 0x745820, let's define it along with our GameAssembly handle now

```cpp
namespace offsets {
    uintptr_t GameAssembly = (uintptr_t)GetModuleHandle("GameAssembly.dll");
    uintptr_t PlayerCharacter = 0x745820;
}
```

---

Take a quick breather because we have got the most difficult stuff out of the way,
all we need to do now is create the hook

<p align="center">I have a less traditional but in my opinion <b><i>better</i></b> way of creating them:</p>

```cpp
void Speedhack() {
    // Directly use gameAssembly as a void*, add the offset, and then cast to void* if needed
    void* targetFunction = reinterpret_cast<void*>(reinterpret_cast<char*>(offsets::GameAssembly) + offsets::playerCharacterUpdate);

    if (globals::speedhack) {
        // Create the update hook
        if (MH_CreateHook(targetFunction, &playerCUpdateDetour, reinterpret_cast<void**>(&playerUpdate_Original)) == MH_OK) {
            // Enable it if it was successfully created.
            MH_EnableHook(targetFunction);
            printf("Hook for PlayerCharacterUpdate enabled successfully.\n");
        }
        else {
            // If debugging is enabled you can see the logs on whether it was successfull or not.
            printf("Failed to create and enable the hook for PlayerCharacterUpdate.\n");
        }
    }
    // Remove the hook after unchecking speedhack || Replace with DisableHook if your game crashes when disabling the hack.
    else if (!globals::speedhack) {
        MH_RemoveHook(targetFunction);
        printf("Disabled unlockAll\n");
    }
}
```

Great! You have almost finished the hack.

Make sure to add the function to the gui and you should be done.

```cpp
if (ImGui::Button("Main")) { tabs::tab = 0; }
ImGui::SameLine();
if (ImGui::Button("Misc")) { tabs::tab = 1; }
ImGui::Separator();

if (tabs::tab == 0) {
	if (ImGui::Checkbox("Enable speedhack", &globals::speedhack)) {
        // The function
		Speedhack();
		printf("\nSpeedhack enabled.");
    }
}
```

---

Before building the cheat, make sure you are on release!

You can build by pressing CTRL + SHIFT + B

<p align="center"><img src="/BuildMode.png" alt="BuildMode.png instruction"></p>

If you encounter any errors with the il2cpp.h file, add an underscore before each thing like so:

<p align="center"><img src="/il22cpp.hErrorFixingUnderscore.png" alt="il22cpp.hErrorFixingUnderscore"></p>

---

To actually inject the hack, you can use [extreme injector](https://github.com/master131/ExtremeInjector/releases/download/v3.7.3/Extreme.Injector.v3.7.3.-.by.master131.rar)

<p align="center"><img src="/EndResult.png" alt="EndResult.png result"></p>

---

That's it, we have done it.

Thank you very much for reading this article, you may have noticed that the farther you got into the article the worse it got and that it because I have spent way too much time on this while also having to do other important things. The next article will be much less sloppy.

***If there is anything wrong, something that I forgot or if you just have any questions I urge you to message me on discord: Basmannetjeee***