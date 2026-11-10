# Penta Dragon DX: decisions needed on issues 55 and 83

Nothing has been merged. Both fixes work, but each one trips a test gate.
I need one decision from you on each.

## Issue 55: yellow garbage before a level loads

**What you see.** When a stage starts, the walls flash flat yellow and the
sprites are white for a moment before the real colors show up.

**Root cause.** The stage fades in using the original Game Boy fade. Our color
system waits until that fade is completely finished before it loads the new
level's colors. During the fade, the screen still uses the previous level's
leftover colors, and one of those leftovers is the flat yellow on the walls.
On the current build that lasts about 46 frames, roughly three quarters of a
second.

**What my fix does.** As soon as the game switches to the new level's map, it
loads the correct background and sprite colors right away, without waiting for
the fade to finish. That cuts the bad colors from 46 frames down to 1.

**Why the tests reject it.** The map switch itself happens inside a piece of
the game's frame-timing code that the title-screen demo also uses. That code
runs almost to the end of the frame, so there is no room for my fix to load
colors before the very first frame of the new level is drawn. That first frame
still shows the old colors, and then frame two is correct. Two tests forbid
exactly this: one says the colors must not change between the stage title card
and the map switch, and one says there can't be a third look on screen after
the switch, meaning an old-colors frame followed by a fixed-colors frame.

**Option A: load the colors inside the shared frame-timing code.**
- Good: it should get rid of the last bad frame completely.
- Cost: it changes code the title demo also runs, and it adds work to the frame
  where the map switches. That risks changing the demo's timing, which we've
  always kept identical to the original, and it needs careful proof that it
  doesn't.

**Option B: relax the stage-card test so it allows this one frame.**
- Good: small and safe. No shared code changes, and the demo is untouched. It
  fixes 45 of the 46 bad frames right away.
- Cost: one frame of old colors stays, about a sixtieth of a second, and we'd be
  loosening a test to fit the fix, which only makes sense if that one frame is
  acceptable to you.

**My recommendation.** Option B now, because it's low risk and removes nearly
all of the flash. If the single leftover frame still bugs you, I'd try Option A
later as its own change.

## Issue 83: garbled text flash before the Stage 1 secret area

**What you see.** Right before the secret area's stage card appears, the map
breaks up into garbled text for a split second.

**Root cause.** It's not the English patch. The Japanese and English builds do
exactly the same thing. To hide the screen while it loads the stage card's font,
the original game sets the black-and-white palette to all black. The Color Game
Boy ignores that palette, so the half-loaded font tiles show up on screen.

**What my fix does.** It does the Color Game Boy equivalent: all the color
palettes go black while the card loads, then the colors come back in a single
frame so the card appears clean. What's left is about six lines at the very top
of one frame. The fix only touches gameplay code, never the title demo.

**Why the tests reject it.** The fix shifts the secret route's timing by a
frame or two. That uncovers a hang that already exists on the current build: a
wait loop in our own color code (bank 20) waits for one exact screen line, and
if a timer interrupt lands on that line every frame, it waits forever. The test
for the BOSS 01 card shows the game freezing there. The hang fix is tiny, just
6 bytes that accept a small range of lines instead of one exact line, with the
same size and the same speed. But three other tests require bank 20 to stay
completely unchanged, so they refuse any edit there.

**Option A: give the 6-byte hang fix a narrow exemption in those three tests,
then merge.**
- Good: fixes the garbled flash and also fixes a real hang that's already in
  the current build, even if it's rare.
- Cost: those three tests need a small carve-out so they treat those 6 bytes as
  belonging to this fix. It's a test-plumbing change, not a gameplay change.

**Option B: hold off.**
- Good: no test changes.
- Cost: the garbled flash stays, and the hang stays hidden in the current build.

**My recommendation.** Option A. The hang is a real bug either way, and I'd also
file it as its own issue so it's tracked.

## Not started yet

Issues 81 (garbled Select menu), 58 (yellow flashing in the boss fight),
82 (room-entry flicker) and 80 (ceiling tearing). Moonshot has been offline,
so it has no new builds yet.

## What to tell me

For 55: "A" or "B". For 83: "A" or "hold off".
