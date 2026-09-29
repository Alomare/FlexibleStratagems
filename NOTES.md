# Stratagems Unleashed (research name: Duplicate Stratagems): research notes

Goal: let a player bring the same stratagem twice (e.g. two Orbital Railcannon Strikes), which the Hellpod loadout
screen normally refuses. Game build 25480438. Addresses are RVAs in game.dll unless marked as Ghidra session
addresses (base 0x7ffcea020000, as in the other mods' notes).

The static analysis used an offline copy of game.dll's code and decompiler output (not included).

## 1. Data (offline)

- StratagemInfo is 400 bytes (typelib). The shipped typelib has no member names, and the community JSON
  (shalzuth/HelldiversData `generated_stratagem_settings.json`) is an older snapshot (62 fields vs 67), so offsets
  come from aligning both by type shape: `+0x0` type, `+0x10` debug_name (char *), `+0x3c` category,
  `+0x50` uses, `+0x64` cooldown_duration_success (f32), `+0x70` origin_type, `+0x94` cooldown_type
  (0 Individual, else SharedTeam / SharedClan), `+0xbc` cost, `+0xc4` depends_on, `+0xc8` additional_stratagem,
  **`+0xcc` max_in_loadout** (u32).
- Stratagem table: pointer array at `0x37cb600` indexed by StratagemType (a default info at `0x37cb470` for 0).
- `max_in_loadout` is 0 for every stratagem except Eagle Rearm and Carpet Bombing Run (1). It is **not** the
  one-of-each rule (see 3).

## 2. The synced loadout is the live mission state

SyncedLoadoutSystem `[0x347ce50]`: player count `+0x2d200`, players at `+0 + i*0x1690`:
- `+0x00` peer id (u64); `+0x38` primary, `+0x98` sidearm, `+0xf8` throwable (weapon blocks of 0x60);
- `+0x1c0` stratagem entries, 48 bytes each, up to 32; `+0x7c0` entry count.
- Entry: `+0x00` StratagemType, `+0x04` uses left, `+0x09` flag byte, `+0x10` cooldown start, `+0x18` cooldown end,
  `+0x20` stamp (u64 microseconds on the scene clock `[0x3326348]+0x18`).

Who writes it:
- `rpc_sync_stratagems` (`0x670530`, `0xb87a70`) and `rpc_sync_stratagem_changes` (`0xb9b040`, `0x135c810`): each
  peer's message carries a count plus up to 16 stratagem types; the builder `0x11e82d0` copies them into that peer's
  entries **as sent**, with no check for repeats. Every peer keeps every player's list.
- Mission start (`0x6ae1f0`): prepends the mission type's stratagems (table `0x213f550`, up to 6 per mission type:
  Reinforce, Resupply, SOS...), then appends each entry's `additional_stratagem`, counting per type and stopping at
  `max_in_loadout` (that is why several Eagles add one Eagle Rearm). Uses come from `0x879550(profile, peer, type)`.
- Stratagem used (`0xb9a6d0(peer, entry index, ...)`, and `0x135c2c0`): starts the cooldown of **that entry
  index** (duration looked up by type through `0x8796a0`). Exceptions: entries whose `additional_stratagem` is 0x31
  (the Eagle family) share the rearm timer; and a type with a shared cooldown type (`+0x94` != 0) is copied to every
  entry of that type in every player's list.

So a list with the same Individual-cooldown stratagem twice should behave as two independent stratagems (own uses,
own cooldown). A duplicated SharedTeam stratagem would share one cooldown (useless but harmless).

## 3. What stops a duplicate

Nothing found in the sync or mission code dedupes the list. The rule seems to be only in the loadout screen
(MenuScreenLoadout, NoesisGUI view models `testament.MenuLoadoutViewModel` / `ScrollListEntryStratagem`), not
traced yet.

The per-player profile record (`[0x33264f8] + 0xa7c + i*0x40`, saved by `0x87b7b0` under MurmurHash64A>>32 keys)
is cosmetic: body_type, helmet/cape/armor, booster (+0xa8c), a 5-entry array (+0xa90, unknown, not stratagems),
victory_pose_id, player_card_index, player_title_index. The stratagem picks live elsewhere (the ship inventory,
per the `ShipInventoryComponent` log lines).

## 3b. The picker's pending loadout (the injection point)

User facts (2026-09-29): a patched glitch once let players fill all 4 slots with one stratagem, and all 4 worked in
missions; and during events that give everyone a free stratagem, players can legitimately bring their own copy too.
The user wants the stratagem picker itself patched (a settings override of filled slots would feel wrong).

The Hellpod loadout screen (`[[0x347ce38] + 176]` when the stack's u32 is 11; ~2.5 MB object, intro timer at
+0x273988) keeps one pending loadout block per player: `screen + 0x10 + i*0x9f0`, local index `screen + 0x27d0`.
**A block has the synced loadout layout**: weapons +0/+0x60/+0xc0, stratagem entries +0x188 (48 bytes), count +0x788.
On exit (`0x1467790`, called from on_exit `0x1467b90` while a sync is pending):
- `0xb4eec0(block)` expands it in place by mission type (mission stratagems `0x6ae1f0`, DSS / event stratagems,
  ...); `screen + 0x72868` points at the local synced list;
- `0x1751350(block)` saves it (stratagem info id + entry +4 per entry, up to 32);
- `0x13620a0(_, peer, block, 1)` sends it to the peers (host, then everyone).
So a duplicate written into the local block before exit goes out the legitimate way. The picker's own rule (which
refuses a stratagem already in the block) is still to be found.

The Noesis `ui.MenuLoadout` page (`xaml/0x4eba493cb089b0cf.xaml`, placeholder art) is not the live picker.

## 3c. Recon 1 (1-recon1)

`research/recon1.lua`, test `research/test_recon1.py` (25 checks on the real dump, mutation-tested). Logs every
change of the four pending blocks while the loadout screen is open (names from StratagemInfo +0x10) and of every
synced list (ship and mission, every 15 frames), dumps the local block on open. F9 (solo lobbies only, build
checked by image size and five prologues): copies the local block's entry 0 over entry 1. F10: marker + dump.

## 3d. Recon 1 results (2026-09-29, solo)

- The local index at screen + 0x27d0 is 0xffffffff when the screen opens and set ~140 frames later (block 0).
- Each pick lands in the local block and, ~10-15 frames later, in the local synced list (the picker syncs on every
  equip). Uses in a block entry: 0xffffffff unlimited, Eagle 500kg 2; flag byte 0 for picks, 1 for mission entries.
- F9 (entry 0 over entry 1) wrote and read back, but the picker's slots on screen did not change and no sync
  followed (nothing polls the block). On exit the block was expanded and sent with the duplicate: mission list =
  Reinforce, SOS, Resupply, Upload Discovery (mission), Precision Strike x2, HMG, Generator Pack, Storm tank
  (mission). **In the mission both Precision Strikes worked, each with its own cooldown (72 s, started 3.6 s
  apart).** So the whole downstream path accepts a duplicate sent the normal way.
- The picker's slot display (and so its "already equipped" rule, which refused the manual re-pick) is separate UI
  state, not the block. Offline leads that were not it: the block-count loop in the screen update `0x1468320`
  (team composition check for VO), the weapon customization senders `0x1034180..0x1034f90`, the launch path
  `0xad9020/0xad91c0` (rebuilds the block from the saved loadout `0x17540b0`), and the Eagle helpers `0x66d650..`.

## 3e. Recon 2 (1-recon2)

Recon 1 plus: F10 anywhere saves numbered raw snapshots (`DuplicateStratagems_snap<N>_screen.bin`, 0x280000 bytes of
the loadout screen object; `_stack.bin`, 0x400 of the screen stack; `_top.bin` when another screen is on top), and
the log records every change of the stack's top type. Plan: snapshots across a scripted pick sequence, diffed offline
to find the picker's slot array and its per-item "equipped" state.

## 3f. Recon 2 results and the picker's rule (snapshots diffed offline)

Sequence (solo): A Precision Strike, B Gatling Barrage, C Airburst Strike, D Napalm Barrage; slot 2's list open
(hovering Railcannon); E Railcannon into slot 2; B back; list open hovering A (refused); F9. The stack's top stayed 11
throughout: the stratagem list lives inside the loadout screen.

- Player panel = screen + 0x53a78 (Transmog's card graph). Its four stratagem slot widgets: panel + 0xea38 + k*0x12a8
  (screen + 0x624b0 + k*0x12a8), type at widget + 0x128c (screen + 0x6373c / 0x649e4 / 0x65c8c / 0x66f34); set by
  `0x1893600(widget, type)` via set_slot `0x189d050(panel, slot, item id)` (item id -> type `0x11f2490`). The slot
  being edited: screen + 0x281c; category: screen + 0x2818 (10 = stratagems). F9 changed the block, not these widgets.
- Equip handler `0x146e0a0(screen, input)`: for a stratagem it maps the item to a type, applies the one-per-kind rule
  (info flags +0x104: 0x100000, 0x200000, 0x400000 = support weapon / backpack / vehicle-like; an equipped one of the
  same kind is replaced in its slot), sets the slot, refreshes the list's markers (`0x18d1890(list, block)`) and saves.
  **It has no duplicate check.**
- Stratagem list = screen + 0xd2f20 (grid screen + 0xd2850, + 0x6d0): offer count +0x92984 (<= 256), offer ids
  +0x92990 (u32), **selectable bytes +0x92dc2**, a second byte array +0x92ec2 (all 0 here). Cells: pages of 0xaed0,
  cells of 0x2b68, flag word at cell + 0x2b5a (bit 2 = equipped look). The refresh `0x18d1890` sets every selectable
  byte to 1, then `0x18d1440(list, offer, 0)` sets 0 (and the look) for each stratagem in the block, mapping item id
  (info +4) to offer id through the offers table `[0x347cef8]` (entries +0xb9ce4, 24 bytes: @4 offer, @8 item; the
  stratagem range: indices at +0xd1d48 from +0xd1d0c to +0xd1d10).
- **The refusal**: the press handler `0x18cf930` asks `0x18d2690(list, page, index)` for the selectable byte and returns
  6 (refused) on 0. Snapshots 2 and 5: 91 offers, exactly the four equipped ones at 0.
- EQUIP prompt/button input `0x191c290` (list + 0x99d70 - 0x6d0): same handler One Click Armor Set met in the Armory.

(RVA correction: functions quoted in chat as 0x17d... are 0x18d... / 0x189...; the list above is correct.)

## 3g. Recon 3 (1-recon3): the picker prototype

Solo lobbies, stratagem category, build checked (prologues plus displacements inside the list functions): each
frame the selectable byte of every list offer that belongs to a stratagem in the local block is set back to 1
(offers of other refused items stay 0). Nothing else is written; the pick then runs the game's own equip path.
Logs the list's refused offers when they change and each kept offer once per screen visit.

## 3h. Recon 3 results (2026-09-29)

- Picking an equipped stratagem again is accepted, the slot icon updates, and in the mission both copies work
  (screenshot: 2x Precision Strike, 2x Autocannon Sentry).
- The list item still draws greyed, no click sound, and the greyed look flickers: the drawing reads the selectable
  byte, and the game's refresh keeps re-greying what recon 3 flipped back.
- Backpacks have no restriction. Mech / FRV / tank: one of each; picking a second of the same kind is redirected by the
  equip handler to the slot already holding that kind (same vehicle: nothing changes; another one: replaces it).
  So the three info-flag bits 0x100000 / 0x200000 / 0x400000 are most likely the three vehicle kinds (they are newer
  than the community JSON's 20 flags). Other code using those masks near +0x104: none tests them on stratagem info the
  way the equip handler does (scan `flag_0x*.txt`; shifted reads would not show).

## 3i. Recon 4 (1-recon4)

- Marks the local loadout's stratagems "not equipped" through the game's marker `0x18d1440(list, offer, 1)` (byte 1,
  look bit cleared, cell redrawn) instead of writing the byte.
- While the stratagem list is open (category 10, list count > 0, solo): clears bits 0x700000 of +0x104 in every
  StratagemInfo that has them (table types 1..0x95), restores them when the list or screen closes or on an error,
  only where the value is still the cleared one. Logs all flags once.

## 3j. Recon 4 results and recon 5 (2026-09-29)

Recon 4: "pretty much flawless" (user): duplicates of every kind, vehicles included, in the loadout, the UI and the
mission; the marker call ungreys and redraws the items. Flags confirmed from the live table: **0x100000 mech**
(Combat Walker, Obsidian, Breacher, Lumberer), **0x200000 FRV** (FRV, Resupply Auto Turret FRV, Ramming Flamethrower
FRV), **0x400000 tank** (Bastion, Storm); 9 stratagems carry them (the President Rewards walker has none). The list
closes after each pick and reopens for the next slot.

Quirk: picking the same stratagem again plays no pick sound. The list's selected offer +0x9298c (set, with a 0x80
highlight bit on its cell, by select `0x18d10d0(list, offer)`; also read by the cell layout `0x18d2b60`, reset by
`0x18d27b0`) is still that stratagem, and the mouse press path only plays equip_sound `0x18d0210(list, offer)` when
the pressed offer differs (`if offer != [+0x9298c] then equip_sound`; the controller path plays it always).
Recon 5: once after each pick (the local block changed), if the selection is a stratagem of the loadout, call
select(list, 0) (clears the field and the highlight the game's way). Not continuous, so controller focus is untouched.

Recon 5 confirmed by the user: the pick sound plays on every repeated pick.

## 3k. Recon 6 (1-recon6): multiplayer

Recon 5 without the solo gate (the list unlock, the selection reset and the vehicle rule run in any lobby; writes stay
local: the local list and the local stratagem table's flags while the local list is open), plus a "Players: N" line
when the count changes. What must hold: the duplicate reaches the other peers through the normal loadout sync (the
receivers copy lists as sent, no dedupe found offline), the host accepts it (host and joiner roles both), unmodded
teammates see and survive it, and both copies work in the mission.

## 4. Open questions

1. Does the game accept a duplicate in practice: HUD shows both, the stratagem input picks the ready copy (both
   share one button code), each keeps its own cooldown, no crash?
2. When is the local player's list filled on the ship, and is it rebuilt at the hellpod launch / mission start?
3. Where the local selection is stored before it is sent, so a duplicate can go out in the sync message itself
   (every peer, including the host, must agree on the list: the call-in likely refers to it by index).
4. The loadout screen's rule, for a native way to pick the duplicate (later; a Mod Options Menu setting like
   "slot 4 copies slot 3" avoids UI work).

## 5. Recon 6 results and release 1 (2026-09-29)

Recon 6 in multiplayer, "flawless" (user): host and joiner, with the teammate unmodded and with both modded;
duplicates (vehicles included) sync, show and work for everyone.

Release 1, **Stratagems Unleashed** (`stratagems_unleashed.lua`, module `mods/alomare/stratagems_unleashed`, same guid
as the research builds; test `tests/test_release.py`, 34 checks, mutation-tested). Kept: the build check (image size,
7 prologues and 19 displacements: the exit sync, the equip handler, the list's selectable check, refresh, press
handling, marker and select), the marker call for refused loadout stratagems, the selection reset after a pick, the
vehicle bits cleared while the list is open. Dropped: all loadout / list / flags logging, snapshots, F10, the player
count. Status: `StratagemsUnleashed_STATUS.log` (OK / NOT AVAILABLE / STOPPED); log `StratagemsUnleashed.log`
(build check and errors only).

Cost (harness counts of ReadProcessMemory): outside the loadout screen one check every 10 frames (0.2 reads per frame);
in the loadout screen on another tab 3 reads per frame; with the stratagem list open 4 reads per frame, one of them
the whole list span (count, selection, offer ids, selectable bytes) into a fixed buffer. The local loadout and the
offers table are read only when the list shows a refused item that was not seen before (after a pick); the offers
map once per screen visit; the vehicle entries once per session (rescanned only if the table changed).

## Version 2: surviving game updates

- Every address and structure offset now comes from the game's code. `research/signatures.py` defines 11 code
  signatures (built and checked for a single match in the dump by `tools/sigspec.py` / `tools/sigtool.py`) and writes
  them into the script. Each is tried at build 25480438's address first (instant on this build), else game.dll's
  executable sections are searched, 4 MB per frame (~15 frames), and it must match exactly once.
- Fields read from the matched code: the screen stack global, the loadout screen slot (+0xb0) and local index
  (`0x1082ef0`: `[[stack]+0xb0]+0x27d0 != -1`); the category (+0x2818), list (+0xd2f20), block base/stride (equip
  handler); list count/selectable/block count/entries/stratagem table, offers global/range/entries and the marker
  call (the list refresh 0x18d1890); ids (marker), selected (select), offers count and the table size / flags
  offset (+0x104) with the kind bits literal (equip handler). Repeated values must agree, within and across
  signatures; the refresh must call the marker found on its own; globals must be in data sections, calls in code.
- Still constants (read-only gates, not in any signature): loadout screen type 11 on the stack, stratagems category
  10. Everything the mod writes or calls depends on values from the code plus those gates plus real offer data.
- Tests: moved code (marker, select, screen query elsewhere; screen slot changed to +0xb8) is found and followed;
  missing, found twice, inconsistent values and a refresh calling elsewhere each turn the mod off. 39 checks;
  engine mutations (first match, field/signature agreement, marker cross-check, fixed slot, natives at the old
  address, chunk overlap counted twice) all caught.
