# Flexible Stratagems (formerly Stratagems Unleashed; research name: Duplicate Stratagems): research notes

Goal: let a player bring the same stratagem twice (e.g. two Orbital Railcannon Strikes), which the Hellpod loadout screen normally refuses. Game build 25480438. Addresses are RVAs in game.dll unless marked as Ghidra session addresses (base 0x7ffcea020000, as in the other mods' notes).

The static analysis used an offline copy of game.dll's code and decompiler output (not included).

## 1. Data (offline)

- StratagemInfo is 400 bytes (typelib). The shipped typelib has no member names, and the community JSON (shalzuth/HelldiversData `generated_stratagem_settings.json`) is an older snapshot (62 fields vs 67), so offsets come from aligning both by type shape: `+0x0` type, `+0x10` debug_name (char *), `+0x3c` category, `+0x50` uses, `+0x64` cooldown_duration_success (f32), `+0x70` origin_type, `+0x94` cooldown_type (0 Individual, else SharedTeam / SharedClan), `+0xbc` cost, `+0xc4` depends_on, `+0xc8` additional_stratagem, **`+0xcc` max_in_loadout** (u32).
- Stratagem table: pointer array at `0x37cb600` indexed by StratagemType (a default info at `0x37cb470` for 0).
- `max_in_loadout` is 0 for every stratagem except Eagle Rearm and Carpet Bombing Run (1). It is **not** the one-of-each rule (see 3).

## 2. The synced loadout is the live mission state

SyncedLoadoutSystem `[0x347ce50]`: player count `+0x2d200`, players at `+0 + i*0x1690`:
- `+0x00` peer id (u64); `+0x38` primary, `+0x98` sidearm, `+0xf8` throwable (weapon blocks of 0x60);
- `+0x1c0` stratagem entries, 48 bytes each, up to 32; `+0x7c0` entry count.
- Entry: `+0x00` StratagemType, `+0x04` uses left, `+0x09` flag byte, `+0x10` cooldown start, `+0x18` cooldown end, `+0x20` stamp (u64 microseconds on the scene clock `[0x3326348]+0x18`).

Who writes it:
- `rpc_sync_stratagems` (`0x670530`, `0xb87a70`) and `rpc_sync_stratagem_changes` (`0xb9b040`, `0x135c810`): each peer's message carries a count plus up to 16 stratagem types; the builder `0x11e82d0` copies them into that peer's entries **as sent**, with no check for repeats. Every peer keeps every player's list.
- Mission start (`0x6ae1f0`): prepends the mission type's stratagems (table `0x213f550`, up to 6 per mission type: Reinforce, Resupply, SOS...), then appends each entry's `additional_stratagem`, counting per type and stopping at `max_in_loadout` (that is why several Eagles add one Eagle Rearm). Uses come from `0x879550(profile, peer, type)`.
- Stratagem used (`0xb9a6d0(peer, entry index, ...)`, and `0x135c2c0`): starts the cooldown of **that entry index** (duration looked up by type through `0x8796a0`). Exceptions: entries whose `additional_stratagem` is 0x31 (the Eagle family) share the rearm timer; and a type with a shared cooldown type (`+0x94` != 0) is copied to every entry of that type in every player's list.

So a list with the same Individual-cooldown stratagem twice should behave as two independent stratagems (own uses, own cooldown). A duplicated SharedTeam stratagem would share one cooldown (useless but harmless).

## 3. What stops a duplicate

Nothing found in the sync or mission code dedupes the list. The rule seems to be only in the loadout screen (MenuScreenLoadout, NoesisGUI view models `testament.MenuLoadoutViewModel` / `ScrollListEntryStratagem`), not traced yet.

The per-player profile record (`[0x33264f8] + 0xa7c + i*0x40`, saved by `0x87b7b0` under MurmurHash64A>>32 keys) is cosmetic: body_type, helmet/cape/armor, booster (+0xa8c), a 5-entry array (+0xa90, unknown, not stratagems), victory_pose_id, player_card_index, player_title_index. The stratagem picks live elsewhere (the ship inventory, per the `ShipInventoryComponent` log lines).

## 3b. The picker's pending loadout (the injection point)

User facts (2026-09-29): a patched glitch once let players fill all 4 slots with one stratagem, and all 4 worked in missions; and during events that give everyone a free stratagem, players can legitimately bring their own copy too. The user wants the stratagem picker itself patched (a settings override of filled slots would feel wrong).

The Hellpod loadout screen (`[[0x347ce38] + 176]` when the stack's u32 is 11; ~2.5 MB object, intro timer at +0x273988) keeps one pending loadout block per player: `screen + 0x10 + i*0x9f0`, local index `screen + 0x27d0`. **A block has the synced loadout layout**: weapons +0/+0x60/+0xc0, stratagem entries +0x188 (48 bytes), count +0x788. On exit (`0x1467790`, called from on_exit `0x1467b90` while a sync is pending):
- `0xb4eec0(block)` expands it in place by mission type (mission stratagems `0x6ae1f0`, DSS / event stratagems, ...); `screen + 0x72868` points at the local synced list;
- `0x1751350(block)` saves it (stratagem info id + entry +4 per entry, up to 32);
- `0x13620a0(_, peer, block, 1)` sends it to the peers (host, then everyone). So a duplicate written into the local block before exit goes out the legitimate way. The picker's own rule (which refuses a stratagem already in the block) is still to be found.

The Noesis `ui.MenuLoadout` page (`xaml/0x4eba493cb089b0cf.xaml`, placeholder art) is not the live picker.

## 3c. Recon 1 (1-recon1)

`research/recon1.lua`, test `research/test_recon1.py` (25 checks on the real dump, mutation-tested). Logs every change of the four pending blocks while the loadout screen is open (names from StratagemInfo +0x10) and of every synced list (ship and mission, every 15 frames), dumps the local block on open. F9 (solo lobbies only, build checked by image size and five prologues): copies the local block's entry 0 over entry 1. F10: marker + dump.

## 3d. Recon 1 results (2026-09-29, solo)

- The local index at screen + 0x27d0 is 0xffffffff when the screen opens and set ~140 frames later (block 0).
- Each pick lands in the local block and, ~10-15 frames later, in the local synced list (the picker syncs on every equip). Uses in a block entry: 0xffffffff unlimited, Eagle 500kg 2; flag byte 0 for picks, 1 for mission entries.
- F9 (entry 0 over entry 1) wrote and read back, but the picker's slots on screen did not change and no sync followed (nothing polls the block). On exit the block was expanded and sent with the duplicate: mission list = Reinforce, SOS, Resupply, Upload Discovery (mission), Precision Strike x2, HMG, Generator Pack, Storm tank (mission). **In the mission both Precision Strikes worked, each with its own cooldown (72 s, started 3.6 s apart).** So the whole downstream path accepts a duplicate sent the normal way.
- The picker's slot display (and so its "already equipped" rule, which refused the manual re-pick) is separate UI state, not the block. Offline leads that were not it: the block-count loop in the screen update `0x1468320` (team composition check for VO), the weapon customization senders `0x1034180..0x1034f90`, the launch path `0xad9020/0xad91c0` (rebuilds the block from the saved loadout `0x17540b0`), and the Eagle helpers `0x66d650..`.

## 3e. Recon 2 (1-recon2)

Recon 1 plus: F10 anywhere saves numbered raw snapshots (`DuplicateStratagems_snap<N>_screen.bin`, 0x280000 bytes of the loadout screen object; `_stack.bin`, 0x400 of the screen stack; `_top.bin` when another screen is on top), and the log records every change of the stack's top type. Plan: snapshots across a scripted pick sequence, diffed offline to find the picker's slot array and its per-item "equipped" state.

## 3f. Recon 2 results and the picker's rule (snapshots diffed offline)

Sequence (solo): A Precision Strike, B Gatling Barrage, C Airburst Strike, D Napalm Barrage; slot 2's list open (hovering Railcannon); E Railcannon into slot 2; B back; list open hovering A (refused); F9. The stack's top stayed 11 throughout: the stratagem list lives inside the loadout screen.

- Player panel = screen + 0x53a78 (Transmog's card graph). Its four stratagem slot widgets: panel + 0xea38 + k*0x12a8 (screen + 0x624b0 + k*0x12a8), type at widget + 0x128c (screen + 0x6373c / 0x649e4 / 0x65c8c / 0x66f34); set by `0x1893600(widget, type)` via set_slot `0x189d050(panel, slot, item id)` (item id -> type `0x11f2490`). The slot being edited: screen + 0x281c; category: screen + 0x2818 (10 = stratagems). F9 changed the block, not these widgets.
- Equip handler `0x146e0a0(screen, input)`: for a stratagem it maps the item to a type, applies the one-per-kind rule (info flags +0x104: 0x100000, 0x200000, 0x400000 = support weapon / backpack / vehicle-like; an equipped one of the same kind is replaced in its slot), sets the slot, refreshes the list's markers (`0x18d1890(list, block)`) and saves. **It has no duplicate check.**
- Stratagem list = screen + 0xd2f20 (grid screen + 0xd2850, + 0x6d0): offer count +0x92984 (<= 256), offer ids +0x92990 (u32), **selectable bytes +0x92dc2**, a second byte array +0x92ec2 (all 0 here). Cells: pages of 0xaed0, cells of 0x2b68, flag word at cell + 0x2b5a (bit 2 = equipped look). The refresh `0x18d1890` sets every selectable byte to 1, then `0x18d1440(list, offer, 0)` sets 0 (and the look) for each stratagem in the block, mapping item id (info +4) to offer id through the offers table `[0x347cef8]` (entries +0xb9ce4, 24 bytes: @4 offer, @8 item; the stratagem range: indices at +0xd1d48 from +0xd1d0c to +0xd1d10).
- **The refusal**: the press handler `0x18cf930` asks `0x18d2690(list, page, index)` for the selectable byte and returns 6 (refused) on 0. Snapshots 2 and 5: 91 offers, exactly the four equipped ones at 0.
- EQUIP prompt/button input `0x191c290` (list + 0x99d70 - 0x6d0): same handler One Click Armor Set met in the Armory.

(RVA correction: functions quoted in chat as 0x17d... are 0x18d... / 0x189...; the list above is correct.)

## 3g. Recon 3 (1-recon3): the picker prototype

Solo lobbies, stratagem category, build checked (prologues plus displacements inside the list functions): each frame the selectable byte of every list offer that belongs to a stratagem in the local block is set back to 1 (offers of other refused items stay 0). Nothing else is written; the pick then runs the game's own equip path. Logs the list's refused offers when they change and each kept offer once per screen visit.

## 3h. Recon 3 results (2026-09-29)

- Picking an equipped stratagem again is accepted, the slot icon updates, and in the mission both copies work (screenshot: 2x Precision Strike, 2x Autocannon Sentry).
- The list item still draws greyed, no click sound, and the greyed look flickers: the drawing reads the selectable byte, and the game's refresh keeps re-greying what recon 3 flipped back.
- Backpacks have no restriction. Mech / FRV / tank: one of each; picking a second of the same kind is redirected by the equip handler to the slot already holding that kind (same vehicle: nothing changes; another one: replaces it). So the three info-flag bits 0x100000 / 0x200000 / 0x400000 are most likely the three vehicle kinds (they are newer than the community JSON's 20 flags). Other code using those masks near +0x104: none tests them on stratagem info the way the equip handler does (scan `flag_0x*.txt`; shifted reads would not show).

## 3i. Recon 4 (1-recon4)

- Marks the local loadout's stratagems "not equipped" through the game's marker `0x18d1440(list, offer, 1)` (byte 1, look bit cleared, cell redrawn) instead of writing the byte.
- While the stratagem list is open (category 10, list count > 0, solo): clears bits 0x700000 of +0x104 in every StratagemInfo that has them (table types 1..0x95), restores them when the list or screen closes or on an error, only where the value is still the cleared one. Logs all flags once.

## 3j. Recon 4 results and recon 5 (2026-09-29)

Recon 4: "pretty much flawless" (user): duplicates of every kind, vehicles included, in the loadout, the UI and the mission; the marker call ungreys and redraws the items. Flags confirmed from the live table: **0x100000 mech** (Combat Walker, Obsidian, Breacher, Lumberer), **0x200000 FRV** (FRV, Resupply Auto Turret FRV, Ramming Flamethrower FRV), **0x400000 tank** (Bastion, Storm); 9 stratagems carry them (the President Rewards walker has none). The list closes after each pick and reopens for the next slot.

Quirk: picking the same stratagem again plays no pick sound. The list's selected offer +0x9298c (set, with a 0x80 highlight bit on its cell, by select `0x18d10d0(list, offer)`; also read by the cell layout `0x18d2b60`, reset by `0x18d27b0`) is still that stratagem, and the mouse press path only plays equip_sound `0x18d0210(list, offer)` when the pressed offer differs (`if offer != [+0x9298c] then equip_sound`; the controller path plays it always). Recon 5: once after each pick (the local block changed), if the selection is a stratagem of the loadout, call select(list, 0) (clears the field and the highlight the game's way). Not continuous, so controller focus is untouched.

Recon 5 confirmed by the user: the pick sound plays on every repeated pick.

## 3k. Recon 6 (1-recon6): multiplayer

Recon 5 without the solo gate (the list unlock, the selection reset and the vehicle rule run in any lobby; writes stay local: the local list and the local stratagem table's flags while the local list is open), plus a "Players: N" line when the count changes. What must hold: the duplicate reaches the other peers through the normal loadout sync (the receivers copy lists as sent, no dedupe found offline), the host accepts it (host and joiner roles both), unmodded teammates see and survive it, and both copies work in the mission.

## 4. Open questions

1. Does the game accept a duplicate in practice: HUD shows both, the stratagem input picks the ready copy (both share one button code), each keeps its own cooldown, no crash?
2. When is the local player's list filled on the ship, and is it rebuilt at the hellpod launch / mission start?
3. Where the local selection is stored before it is sent, so a duplicate can go out in the sync message itself (every peer, including the host, must agree on the list: the call-in likely refers to it by index).
4. The loadout screen's rule, for a native way to pick the duplicate (later; a Mod Options Menu setting like "slot 4 copies slot 3" avoids UI work).

## 5. Recon 6 results and release 1 (2026-09-29)

Recon 6 in multiplayer, "flawless" (user): host and joiner, with the teammate unmodded and with both modded; duplicates (vehicles included) sync, show and work for everyone.

Release 1, **Stratagems Unleashed** (`stratagems_unleashed.lua`, module `mods/alomare/stratagems_unleashed`, same guid as the research builds; test `tests/test_release.py`, 34 checks, mutation-tested). Kept: the build check (image size, 7 prologues and 19 displacements: the exit sync, the equip handler, the list's selectable check, refresh, press handling, marker and select), the marker call for refused loadout stratagems, the selection reset after a pick, the vehicle bits cleared while the list is open. Dropped: all loadout / list / flags logging, snapshots, F10, the player count. Status: `StratagemsUnleashed_STATUS.log` (OK / NOT AVAILABLE / STOPPED); log `StratagemsUnleashed.log` (build check and errors only).

Cost (harness counts of ReadProcessMemory): outside the loadout screen one check every 10 frames (0.2 reads per frame); in the loadout screen on another tab 3 reads per frame; with the stratagem list open 4 reads per frame, one of them the whole list span (count, selection, offer ids, selectable bytes) into a fixed buffer. The local loadout and the offers table are read only when the list shows a refused item that was not seen before (after a pick); the offers map once per screen visit; the vehicle entries once per session (rescanned only if the table changed).

## Version 2: surviving game updates

- Every address and structure offset now comes from the game's code. `research/signatures.py` defines 11 code signatures (built and checked for a single match in the dump by `tools/sigspec.py` / `tools/sigtool.py`) and writes them into the script. Each is tried at build 25480438's address first (instant on this build), else game.dll's executable sections are searched, 4 MB per frame (~15 frames), and it must match exactly once.
- Fields read from the matched code: the screen stack global, the loadout screen slot (+0xb0) and local index (`0x1082ef0`: `[[stack]+0xb0]+0x27d0 != -1`); the category (+0x2818), list (+0xd2f20), block base/stride (equip handler); list count/selectable/block count/entries/stratagem table, offers global/range/entries and the marker call (the list refresh 0x18d1890); ids (marker), selected (select), offers count and the table size / flags offset (+0x104) with the kind bits literal (equip handler). Repeated values must agree, within and across signatures; the refresh must call the marker found on its own; globals must be in data sections, calls in code.
- Still constants (read-only gates, not in any signature): loadout screen type 11 on the stack, stratagems category 10. Everything the mod writes or calls depends on values from the code plus those gates plus real offer data.
- Tests: moved code (marker, select, screen query elsewhere; screen slot changed to +0xb8) is found and followed; missing, found twice, inconsistent values and a refresh calling elsewhere each turn the mod off. 39 checks; engine mutations (first match, field/signature agreement, marker cross-check, fixed slot, natives at the old address, chunk overlap counted twice) all caught.

## Version 3 (recon 1): Flexible Stratagems, rulesets, copy cooldown

Renamed: module `mods/alomare/flexible_stratagems`, entry `flexible_stratagems.lua`, global `FlexibleStratagems`, logs `FlexibleStratagems*.log`, Mod Options Menu option `alomare.flexible_stratagems.ruleset`; the guid is unchanged.

Rulesets (Off / Less Restricted / Unleashed, default Less Restricted, also without Mod Options Menu):
- Off: nothing is marked or written; the vehicle bits are given back at once when chosen.
- Less Restricted: the marker only puts back stratagems that are in the loadout once (a second copy stays refused). The vehicle kind bits are lifted as in Unleashed, so the copy limit applies per stratagem: two of the same tank are allowed, and so are two different tanks (each is one copy of itself). Confirmed by the user (2026-10-03).
- DiverKit warns when a loadout with duplicates is saved; it records the preset but may refuse to apply it (user, kept as a note on the Nexus page).
- Unleashed: version 2.

Copy cooldown (offline, the use handler 0xb9a6d0 / 0x135c2c0):
- Synced loadout system `[0x347ce50]`: players `+0x2d200`, 0x1690 bytes each, peer id at +0; stratagem entries at `+0x1c0` (48 bytes: type +0, uses +4, cooldown start +0x10, end +0x18, base +0x20), count `+0x7c0`. Times are scene clock microseconds (`[0x3326348] + 0x18`).
- Using entry k: start = now (when flagged), end = duration * 1e6 + base. The Eagle family (type 0x31 Eagle Rearm, or info +0xc8 == 0x31) copies the rearm to every Eagle entry; a shared-cooldown stratagem (info +0x94 != 0) copies end, start and base to every entry of its type in every player's list.
- The mod: every 6 frames, the local list (peer id `[0x347cef0] + 0xb398`, getter 0x5bae80), one read of the entries; an entry whose end moved past now started a cooldown, so the other entries of its type (unless Eagle family or shared) are held: start = base = now, end = now + 10 s, written again whenever lower (a host sync) until the 10 s pass. A different list (mission start, loadout change) starts over without comparing.
- Live test of 3-recon-1 (2026-10-04, solo and in a squad): the three rulesets, the copy cooldown (a 4-player session's list "2 of 4" included) and the Eagle and shared-cooldown exceptions all worked as intended, so the client-side cooldown holds. Released as V3.
- Signatures (optional; without them only the copy cooldown is off): `cooldown_copy` 0xb9a983 (the shared-cooldown loop: synced global, all entry offsets, +0x94), `cooldown_start` 0xb9a7cc (clock, +0x18, start), `eagle_family` 0xb9a8af (+0xc8, 0x31), `self_peer` 0x5bae80 (session global, +0xb398).

Texts: `locales/en.lua` + `locales/pt-BR.lua` (bingus_text); OFF is passed as the plain English word.

## Version 4 (recon 2): copies slider, ready with empty slots, list kept open (2026-10-05)

User requests: drop the copy cooldown and the rulesets for a 2-4 copies slider (vehicles always lifted); allow readying up with fewer than 4 stratagems; after a replacement in a full loadout, keep the list open on the next slot instead of closing it.

Copies slider: Mod Options Menu slider `alomare.flexible_stratagems.max_copies` (2 to 4, default 2; a new id, since the old `ruleset` held a choice index). The marker puts back a loadout stratagem while it has fewer copies than the limit; at 4 the limit never binds (4 slots). The cooldown code and its four signatures (`cooldown_copy`, `cooldown_start`, `eagle_family`, `self_peer`) are removed.

The ready gate (offline, `_research/fewer/`): the READY UP prompt (input action `ReadyUp`, settings id 0x35, menu input 0x24) is the local panel's prompt at panel + 0x28b0; the screen (`0x146d9b0`) calls the panel's ready handler `0x189c250(screen + 0x53a78, input)` while the stratagem list is closed (screen + 0x273990 == 0). Panel 0 is the local one (the equip handler also uses it). The handler:
- counts the four slot widgets (flags u16 at panel + 0xfcc0 + k * 0x12a8, type at +4): filled = type != 0; disabled = empty with flag bit 2 (bit 2 is the widget's disabled flag: no hover, so it can't be clicked); owned = stratagem types 1..0x95 passing `0x136fc20(offers, type)`;
- required = min(owned, 4 - disabled); while filled < required it flashes each empty enabled slot (an animation, and byte +0x18 set to 1, cleared by the widget update `0x18932c0` when the animation ends), plays 0xf50f0501 and returns 5. This is the "pick 4 (or all you own)" rule. The string "You need to select four stratagems to be able to ready up" (0xb23162b7) is unused;
- else toggles: for the local panel (+0x1ee0c), when the timer (+0x1ee14) is -1.0 it starts at 1.75 s with sound 0x7947920 (every other player ready: bit 3 or 11 of players + 0x3ac + i * 0x20) or 0x4d777731; the panel update (`0x189b920`) counts it down and then sets bit 3 of the local player's flags (players `[0x3326468]` + 0x3ac, when +0x88 is set and the first entry's byte +0x14 has bit 0) — the ready state. Otherwise the timer goes back to -1.0, bit 3 is cleared and 0xf50f0501 plays. The UI sound function is `0x1327f50(unused, id)`; `0xbf2e20` is only telemetry.

Approach (no code patch: game.dll is Themida-packed): the mod watches the local panel's four flash bytes while the list is closed; a 0 -> 1 change is the handler refusing a ready press, and the mod then runs the toggle itself (timer and sound, or the cancel). The refusal's own sound (0xf50f0501) and flash still play first. The flash bytes are left alone (zeroing one early would skip the widget's end-of-flash call `0x18932f0`), so a second press while a slot still flashes is not seen; the next one after the flash is. Rejected: setting the disabled bit on empty slots (the player could no longer click them) and setting it only while the Ready key is down (depends on whether the Lua update runs before or after the UI update in a frame).

The list kept open: after an equip the handler (`0x146e0a0` tail, `0x146e609`) looks for the first enabled empty slot (types at screen + 0x6373c, not in grid mode 1 at +0x6f290); with one it calls the grid's focus setter `0x1895770(screen + 0x595f0, slot)` (focus at grid + 0xd96c = screen + 0x66f5c) and sets the edited slot (+0x281c), keeping the list open; else it plays 0x97753411 and closes the list (`0x146f3b0`). The opener `0x146e9d0(screen)` opens the list for the focused slot. The mod remembers the edited slot and the loadout while the list is open; when it closes after a pick (the loadout changed) that kept 4 stratagems (a replacement, not the last fill) from slot 1 to 3, it calls focus(slot + 1) and the opener. Escape (no change), slot 4 and the last fill close as before.

Signatures (optional; each feature turns off alone): `ready_call` 0x146d9fb (list_open, panels, the handler), `ready_slots` 0x189c3b8 (slot flags, flash, stride, UI sound), `ready_toggle` 0x189c4d0 (players, flags, timer, times, sounds; must lie inside the handler), `equip_tail` 0x146e609 (grid mode, slot types = panels + slot flags + 4, grid, focus call = the setter found, edited slot), `open_list` 0x146e9d0, `focus` 0x1895770.

Tests: 67 checks; mutations of the flash edge, the replacement test (4 before and after), the timer test and the slot 4 limit are caught.

Questions for the test: does Ready with 1-3 stratagems ready you up (and cancel on a second press), with teammates seeing you ready and the mission starting with your short loadout (host and client)? How do the error buzz and slot flash before the ready sound feel? After replacing slot 1-3 of a full loadout, does the list reopen on the next slot, with picks working normally there?

## 4-recon-2 results and 4-recon-3 (2026-10-05)

Live test of 4-recon-2: the slider, ready with fewer stratagems (the mission starts with the short loadout) and the reopened list all work. User requests: remove the red flash before readying; when the list reopens it jumps to the stratagem already in the next slot instead of staying where the player scrolled; add a Clear Stratagems key that empties all 4 slots.

No flash (offline, `_research/fewer/`): the handler only counts empty slots that are not disabled (flag bit 2), and runs only when its READY prompt (panel + 0x28b0) fires. The prompt's press check `0x1891850(prompt, input)`: nothing when +0x2f84 == 1 (state 1 swaps the prompt for another widget, 2 hides both); else the action at +0x2f88 (index = high dword + low dword * 0x61 for a low dword up to 12, as `0x585b80` maps it: ReadyUp 0x3500000000 with mouse and keyboard, 0x300000003 on a controller) pressed this frame: byte input + 0x328 + index * 0x20; or confirm (index 0xa) while hovered (+0x2f90) in mouse mode (input + 0xe1ad4) or focused (+0x2f91) otherwise. The input object is the global `[0x347cf18]` (the same one the mission return's skip input reads; ImpatientDiver's NOTES quote it as 0x349cf18, a Ghidra address slip). When the prompt fires with the list closed, the mod sets bit 2 on the empty enabled slots and clears it the next frame (only where its value is still ours): the handler then sees required <= filled and readies with its own sound. Whether the Lua update runs before the UI update in a frame is not known offline: if after, the handler refused first and the flash fallback readies (the log says "The handler refused before the mod saw the press"). Bit 2 for one frame can drop a hover on an empty slot for that frame. Rejected: zeroing the flash bytes (skips the widget's end-of-flash call), the prompt state (swaps the button), the owned count (the offers table, also the list's).

The list's scroll: the opener rebuilds the list (`0x18d8710(list grid, item of the focused slot, block)`), which resets it and focuses the slot's current item (or the first selectable cell when empty): the jump. The list is smooth-scrolled: float +0x92960 (clamped to [0, +0x8c0] by the list update `0x18cf820`), laid out by `0x18d2b60(list)`; `0x18d1280(list, offer)` focuses a cell without scrolling. The mod remembers the scroll while the list is open; after reopening it focuses the stratagem just picked (the edited slot's widget type -> item -> offer), writes the scroll back and lays the list out.

Clear Stratagems: a pick runs `0x189d050(panel, slot, item)` (slot widget = panel + 0x5b78 + 0x8ec0 + slot * 0x12a8; the widget setter `0x1893600(widget, type)`, type 0 hides the icon), then `0x189d120(panel, -1)`, which writes the slots into the loadout block through `0x18966a0(grid, [panel + 0x1edf0])` (clears the entries, then appends the non-empty slots) and sends the change; the equip tail then refreshes the list and saves the block (`0x1751350(screen + 0x10 + index * 0x9f0)`). The key (Mod Bindings Menu `alomare.flexible_stratagems.clear`, "Clear Stratagems") does the same with type 0 for every filled slot: widget setter, `0x189d120`, save; with the list open it closes it first (`0x146f3b0`); not while readying or ready. The item -> type mapping `0x11f2490` is not used (Ghidra's listing labels it 0x1212490).

New signatures (optional): `ready_entry` 0x189c25a (the prompt offset; its press check must be `ready_press`), `ready_press` 0x1891850, `set_slot` 0x189d050 (the grid offset 0x5b78 stays literal: a twin setter for another grid differs only there), `list_update` 0x18cf84f, `list_focus` 0x18d1280; `equip_tail` widened to 0x146e5f6 (slots written, list and grid refresh, save, block layout).

Tests: 81 checks. The harness runs the game's ready handler after the mod's frame and before it; mutations of the mark release, the inactive prompt, the confirm rule, the empty test, the block write and the scroll write are caught.

Questions for the test: does Ready with empty slots ready up without the red flash (the log says whether the handler refused first)? Does the reopened list stay where you scrolled? Does Clear Stratagems empty the slots, and does the cleared loadout stay cleared after leaving the screen and in a squad?

## 4-recon-3 results and release 4 (2026-10-05)

Live test of 4-recon-3: the kept scroll (the picked stratagem focused) and Clear Stratagems work. The ready press seen before the game does not: the log shows "Ready press with 3 empty slot(s): held as disabled", "The handler refused before the mod saw the press" and "Ready with 3 empty slot(s)" on the same frame, so the game's UI update runs before the Lua update in a frame and the handler always sees the press first. The user kept the flash (the game noticing the empty slots), so the press path and its signatures (`ready_entry`, `ready_press`) are removed: ready with empty slots works through the flash alone.

Release 4: 20 signatures; tests 73 checks (list closed: 8 reads per frame). Released as V4.

## 4-fix-1: un-ready with empty slots (2026-10-05)

User report on V4: after readying with fewer than 4 stratagems, un-readying takes repeated presses for ~2 s. Cause (offline): the handler only flashes a slot whose flash byte is 0 (`if flash == 0 and not disabled then animate; flash = 1`), but still refuses (error sound, returns 5) while one is flashing; the widget update `0x18932c0` clears the byte only when the animation ends. The mod acted only on a flash starting (0 -> 1), so every press during the flash of the previous press was refused by the game and unseen by the mod.

Fix: while a flash shows, the mod asks the game's own press check `0x1891850(panel + 0x28b0, input)` whether the READY prompt fired this frame. The check only reads (prompt state +0x2f84, action +0x2f88 through the mapper `0x585b80`, input + 0x328 + index * 0x20, confirm while hovered / focused by input mode `[input] + 0xe1ad4`). The input object is `[0x347cf18]`: the UI frame (`0x1437860`) loads it and passes it to the stack dispatcher `0x14ac950`, which for screen type 11 calls the loadout screen's input `0x146d370`, then `0x146d9b0`, then the handler. Pressed + a slot flashing + the timer's idle state unchanged since last frame (the handler's own toggle always flips it: idle -1.0 <-> 1.75 s; the panel update's countdown leaves it at <= 0, never -1.0) = refused, and the mod toggles. The flash-start edge stays as before (also guarded by the timer). The many gates in `0x146d370` before the handler are not mirrored: a press the screen ignores while a flash from a refusal < 2 s ago still shows would be toggled (a narrow window, e.g. the launch starting).

Signatures (optional; without them the press during a flash is off and the V4 behavior remains): `ready_entry` 0x189c25a (panel_entity, panel_local, ready_prompt 0x28b0, press_call; must lie at most 0x40 into the handler, and its call must be `ready_press`), `ready_press` 0x1891850 (the input global). Cost: list closed, 9 reads per frame (the timer added); the press check is called only while a slot flashes.

Tests: 83 checks; mutations of the timer guard, the press path and the prompt offset are caught.

Question for the test: ready with 1-3 stratagems, then press Ready again right away (during the flash): does it cancel on the first press, and ready again on the next?

## 4-fix-1 results and 4-fix-2: the ready pose (2026-10-05)

4-fix-1 confirmed live: ready and un-ready with empty slots work on the first press. Remaining gap: no ready pose (the salute) when readying with fewer stratagems, locally or (presumably) for the squad.

Cause: the handler's toggle (`0x189c250`) does one thing the mod's toggle skipped: before writing the timer it calls the emote sender `0xbf2e20(unused, unit = [panel + 0x1edfc], event, 0.0, 0.0)` with event `0x97e150a4` when readying and `0x5d1d1963` when cancelling. The sender packs the four values (two uints, two floats) and sends RPC `0xf24760a9` to all peers (`0xbde430(rpc, -1, args, 4)`); the panel reset `0x189dde0` and the screen's reset `0x1470700` send the same stop event (skipping a unit of -1). The countdown in the panel update `0x189b920` sends nothing. So the pose is the RPC alone, the same path the vanilla ready takes for every player.

Fix: the mod's toggle sends the same event through the same function, in the same order (before the timer write), and skips a unit of -1 like the reset does.

Signatures (optional; without them the pose is off and the ready still works): `ready_emote` 0x189c552 (inside `ready_toggle`, at most 0x100 past it: panel_unit, emote_cancel, emote_ready, the emote call; ready_timer, players and ui_sound agree with the other ready signatures), `emote_send` 0xbf2e20 (the RPC id and the send call; the `-1` stays literal). The toggle's emote call must be `emote_send`, and the unit offset must lie between the entity (+0x1edf8) and the timer (+0x1ee14). No extra reads per frame; one read and one call per mod toggle.

Tests: 88 checks; mutations (no pose on ready / on cancel, events swapped, no -1 guard, wrong unit offset, pose always off) are caught.

Question for the test: ready with 1-3 stratagems: does the Helldiver salute, and does un-readying stop it? With a squad: do the others see it?

## 4-fix-2 results and release 5 (2026-10-05)

4-fix-2 confirmed live: readying with fewer stratagems plays the ready pose. Released as V5 (both fixes).
