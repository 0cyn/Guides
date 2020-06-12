Note:
* All discussion of resources that can be obtained from InternalUI builds should be regarded as hypothetical and purely educational. Obtaining these builds without the express permission of Apple is illegal, and doing so is discouraged. All information provided here is purely educational. 
* This guide was written for IDA 7.5. It should work on 7.3 and above. If you're using a cracked version, Scroll down to the "Pre 7.3" section. The rest doesn't apply to you at all.

# Crucial Performance Tips
* Close the function window while analyzing to speed up processing about 10 times, typically
    * An active filter in the functions window will hurt speed much worse
* When analyzing *massive* files, close all of the windows inside IDA (IDA View-A, functions, output, etc). Processing will speed up anywhere from 5x to 100x
* Disable Lumina. Find a way to delete it from IDA if you can. If enabled, it will irreversibly "fix" names by setting them to completely incorrect values. This will waste your time and hurt your analysis.
* IDA seems to operate on one thread. This means when doing anything remotely intensive, it will appear to freeze. This is normal. Don't Force Quit. 
* *heavily* consider maxing out your RAM if you work with IDA very often. You won't regret it, and it will avoid system crashes when working with multiple shared caches.
* If you're on an OS with the ability to create "desktops", I'd highly suggest you give IDA it's own. It will block your UI while loading otherwise.

General tips regarding IDA usage for iOS RE:
* If you are not patient, do not use IDA on the dyld_shared_cache. You will lose your mind.
* Modern versions of IDA come with a dark mode included. Google "IDASkins" if you are on an older version and enable a dark mode. Your eyes will thank you if you work at night. 

A majority of the information in this article details the process of reverse engineering using the dyld_shared_cache, as doing such is poorly documented in official documents.


# Terms used 

* "Module" represents a Framework or library located in the dyld_shared_cache
* "Segment" or "Module Segment" refers to a specific segment of a framework. 
    * The text segment contains code
    * IDA 7.3 and greater includes the ability to load only data segments on-demand without processing the text segment. This is huge.

# Analyzing the dyld_shared_cache in IDA Pro 7.3 or later.

IDA 7.3 and later includes a powerful, improved shared cache toolkit. It eliminates the need for simulator binaries, and makes analysis possible when you cant get access to simulator binaries (InternalUI builds, no macOS, no x64 decompiler, etc.)

The documentation is not great, and as such, I've made an attempt at documenting my own experience with the software.

Everything described here was performed on a licensed copy of IDA Pro 7.5. Older, especially unlicensed versions, may not be able to handle all of these features.

---

## Analyzing a specific framework from the dyld_shared_cache. 

**Do not "Load module and dependencies" option on "high level" frameworks.** In iOS 13, with `SpringBoardHome` this results in loading 720 modules. This takes upwards of 2 to 3 days on an 8-core 4GHz 32GB-of-ram PC. In newer versions, due to consolidation, that number is down to ~400. You'll still be unable to use your PC for a few days at best. I have loaded an entire shared cache a total of 3 times. I could write a separate article on the unfixable issues that happen. It's not worth your time, I promise. Utilize the tools described below.

IDA 7.3 introduced powerful new tools for dealing with the cache. You can now load a single module and selectively load only segments you need from other locations in the shared cache. It can be a pain, but the alternative is much, much worse.

### Load the framework you're interested in

1. Select the "Load single module" option. Ensure you do not select "with dependencies". 
2. Wait for the module you selected to load. It shouldn't take long. 

For this example we'll be using FrontBoard.framework. 

Loading is the easy part. Now we get to go through the process of correcting IDA's failures, as certain functions tend to fall apart in the dyld_shared_cache subsystem.

### Troubleshooting missing data (red addresses, garbage variable names, etc)

The first thing you'll notice is that the assembly or pseudocode generated is absolute gibberish. If regular assembly is gibberish to you, this is *advanced* gibberish. 

Swap to the IDA view for this. You may not be able to read assembly, but the pseudocode view doesn't properly handle the new features.

#### Red addresses

Swap to the "IDA View", as it doesn't work properly in the pseudocode view, and right-click a red address. We are going to assume that the one you clicked was a reference to libobjc.dylib, although it could be any library or framework in the cache.

You'll see an option to load "libobjc.A:__OBJC_RO" or something similar, or an option to load the entirety of "libobjc.A". If you don't need to reverse the contents of "libobjc.A" (you don't), you should simply load only the segment IDA suggests. This allows you to avoid absolutely destroying your RAM and CPU when working in the cache, while also allowing you to make sense of the code within it. 

#### If the address is still red:

IDA likely failed to recognize any information in the segment. This can be caused by a damaged database, if IDA crashed while processing data.

Click the address and you'll be taken to the memory location, and if that assumption is correct and you can see vertical strings of letters:

* In the Menu, click Edit -> Other -> Objective C -> Reload Objective C Info, then navigate to the function you're disassembling
* Right-click the red address and re-load the segment. Check "Do not show again for this session" unless you like repetition.
* In the Menu, click Edit -> Other -> Objective C -> Reload Objective C Info (yes, again)
* Right-click the far left string on the IDA View that holds the name of your target framework.
* Load the entire framework (2nd option), not a specific section.
* Right-click the red address and load the segment again. 

Your address is probably still red. If so, you've damaged your database. I'd advise deleting the database and starting from scratch. This is the fastest option.

#### off_xxxxxxxxx (random hex address prefixed by "off_") in your assembly

**What causes this?**

These represent "refs". You're most likely looking at a class ref that failed to load.

**Fix**

1. In the IDA View, double-click the `off_x` variable to be taken to the classrefs segment
2. Right-click the red memory address and load the suggested module segment.

A name will appear. Good. Go back to your function.

3. Edit -> Other -> Objective C -> Reload Objective C Info

If it changes from `off_x` to `selRef`, `classRef`, or something similar, you can move on.

If it does not change, see below

**What causes this?**

IDA improperly guessed the type of a struct it loaded due to a missing segment.

**Fix**

1. Double click the pink text if you haven't yet to be taken to the class definition in __objc_data
2. Click the `_OBJC_CLASS...` item to select that line
3. Open Edit -> Struct Var
4. Select objc_class and hit OK
5. A red memory address will appear. Load that segment.
6. Make your way back to your function that you're disassembling. 
7. Edit -> Other -> Objective C -> Reload Objective C Info
8. Cry, because it's finally fixed.

Repeat this for any variables you feel are worth spending the time correcting.

#### Other issues

I'm likely forgetting some. I loaded a shared cache fresh and walked myself through fixing issues for the sake of this guide. I'll continue to add solutions as I encounter them.

---

Interesting Note:
Sometimes, you'll see an address and click to load it in. "What on earth is 'GeoServices' doing in this function?" You might think. Upon loading, you'll see it was something like `j__objc_retainAutoreleasedReturnValue_0`. This is a byproduct of the shared_cache's optimizations, and as a result, you'll end up with several duplicate functions like this. A script to fix these needs to be written, eventually. 

---

Typically loaded frameworks:  
libobjc  
Foundation  
CoreFoundation  
GeoServices ("trampoline")

I'm very interested in the concept of creating a "template database" that has data segments for these and others pre-loaded. If someone tries that, do update here with how that's done best.

---


## Working with pseudocode from the dyld_shared_cache

Something you'll likely become familiar with is the statement `(self + 10)`, where 10 is any 2 digit number. In objc source, you would see this as an ivar. If you've loaded in the relevant class information, you can help IDA display these ivars properly in the pseudocode view like so:

* Right-click the `a1` or `self` variable on line one
* Click Y or "Set ivar type"
* Change the class of `self`/`a1` to the class shown next to it.
* Change a1 to self if need be

Ivars should now be properly generated and shown in pseudocode.

---

## InternalUI .development cache

While someone more experienced could speak to the exact purpose of this build of the cache, given that a dump was leaked to the general public I see no need not to discuss this.

The .development cache (which cannot be loaded in cracked IDA versions) appears to be a build of the shared cache that properly holds symbols for the libobjc, libsystem, and other libraries, instead of raw addresses.

If you're using Hopper or IDA 7.5, give the .development cache a shot.

Do note, I've had some issues with certain functions in it. I'm excited to see more information or research on the functionality of this object.

#### Other fun easter eggs in dumps

I'll leave these for you to find, but as a hint, look in folders that normally have no binaries and you might find a nice treat. 

(not to mention the kexts, who needs kernelcaches anyways)

---

# Pre-7.3 dyld_shared_cache analysis

I do not intend to pick a fight with illegitimate IDA users. The software is insanely expensive, I cannot fault anyone in that regard. If you are using an illegitimate copy, don't tell me, I don't want to know. Best of luck.

It's from a year of experience with it that I'm telling you:
* Illegitimate versions of IDA cannot properly handle arm64e code very well.
* Illegitimate versions of IDA cannot properly handle .development versions of shared caches available in InternalUI dumps whatsoever. It's completely incapable, and fails to process modules it loads. 
* Users of illegitimate versions of IDA should primarily stick to Simulator runtime binaries as detailed below. 
* Consider Hopper. It is capable of a few of IDA >7.3's features (arm64e, .development caches) and carries a much smaller price tag
    * Get comfortable with assembly if you intend to use Hopper. The pseudocode it generates is among the "least desirable" in the industry, and the assembly is easier to read.
* Additionally, consider Ghidra. I'm not familiar with it, but others are, and can help you work with it. 
    * Ghidra pseudocode I have heard is on par with IDA 7.0's pseudocode.

I've decided to leave the below sections in this guide for educational purposes, but using a cracked IDA here is more than likely a waste of your time compared to the myriad of options available. Additionally, I obviously cannot condone the usage of such. 

## If you are dead-set on using the arm64 shared cache:

**Before you start analyzing the entire thing, I've already done that!** I've publicly shared the fully processed cache here: https://developer.openpack.io/dyld_shared_cache_arm64.i64. Do not let my sacrifice of 4 days be in vain. Use this, don't waste your time. 

It includes SpringBoardHome and 740 other frameworks SpringBoardHome depends on. It's a 13 GB file. Have fun.

This is not worth it, you have been warned. *Please* consider using simulator binaries instead. 

**Trying to search for a function name will crash IDA entirely**. Close the Functions view and open the "Program Segmentation" window. Browse frameworks like this, and carefully scroll through to find the function you want. 

Although you can work around the Function name search crash by using a full filter instead of the quick filter, this will cause decompilation to take several minutes while the filter is active. Additionally applying and removing the filter will take several minutes (but typically doesn't crash). 

## Simulator Binaries: the recommended solution on older IDA versions

The iOS simulator runtime is for you. x64 binaries that don't have the "Red Address" issue are available.

Find them here: /Library/Developer/CoreSimulator/Profiles/Runtimes/iOS\ 12.4.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks

You may need to change the name of the folder for the simulator versions you have downloaded and installed.

# dsc_fix.py Plugin

This plugin no longer functions, as the IDA SDK no longer provides the needed interfaces. Additionally, it needs to be updated for python3. Probably works on old IDA versions, do let me know.
