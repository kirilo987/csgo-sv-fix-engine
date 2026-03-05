If you have a CS:GO (NOT CS2) server, then users cant join your server IF they are using the archive of CS:GO (not csgo\_legacy from CS2 BETAS): app/4465480/CounterStrikeGlobal\_Offensive/
And IF you would want to allow users to join your server using THIS CS:GO archive, then follow my tutorial :D

!THIS ONLY WORKS IF YOU ARE THE SERVER OWNER, WITH ACCESS TO EVERYTHING!

STORY N TUTORIAL;

Valve archived CS:GO and made it available as a separate Steam app, and the problem is that clients using the archived build **cant connect to community servers**.
The auth ticket comes in with a mismatched appid and the server rejects it:

    S3: Client connected with ticket for the wrong game
    RejectConnection: STEAM validation rejected

**Root cause**
The engine has a jump table dispatch that handles appid validation. Case 4 (archived build) hits a rejection path instead of the default OK path.

**Easy fix (prebuilt .so (shared lib)):**
I made a shared lib that fixes it automatically, you can load it via `LD_PRELOAD`:

    LD_PRELOAD=/home/container/csgo_steamfix.so ./srcds_run -game csgo

If it works, you'll see something like this in console:

    [steamfix] made by E0n3x!
    [steamfix] waiting for engine..
    [steamfix] sig at 0xf359b2ba (vaddr=0x18a2ba)
    [steamfix] jt=0xf3912b64 jt[0]=0xf359b498 jt[4]=0xf359b3d8
    [steamfix] patched engine!

After this, **everyone can connect,** archived build clients included.
**You can download the prebuilt lib from the repo**

**Manual fix** *(only if you know reverse engineering):*
Find the jmp dispatch via IDA (sigmaker):

    FF 24 85 ? ? ? ? 8D B4 26 ? ? ? ? 31 F6

That's `jmp ds:jpt[eax*4]` , then open the jump table at `.rodata:jpt_18A2BA`:

* `[0]` → default (status OK)
* `[4]` → loc\_18A3D8 ← this is the rejection path

Solution: copy `jt[0]` into `jt[4]` at runtime. (case 4 then falls into the OK path)
The table address is embedded in the instruction itself (`FF 24 85 [addr]`), so no separate sig needed for the table.

>Should work on any engine version, just find the jmp dispatch and identify which table index maps to the wrong-game rejection case.
