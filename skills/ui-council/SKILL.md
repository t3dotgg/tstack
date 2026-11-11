---
name: ui-council
description: Trigger this if the user says 'ui council', and before shipping a new page, redesign or visual overhaul.
---

# UI Council

Judge big UI changes against the apps they copy until they beat them. Small fixes
skip this.

1. Screenshot the real app with realistic data: desktop and phone, light and
   dark. Move the mouse away first.
2. Get the matching reference screens from Mobbin.
3. Make a folder: `pairs/` holds one of yours and one reference per pair,
   named `a` and `b` in random order; `screens/` holds the rest of yours. Keep
   the a/b key outside the folder.
4. Run the `council` skill on it. Each delegate scores both sides of every pair
   and the whole set out of 10, and lists at most 8 fixes.
5. Fix what two or more delegates ask for in the same round. Skip one-off
   asks and anything a delegate reversed.
6. Repeat until every delegate rates the set 8.6/10 or more and yours wins every
   pair. Only the user ends it early.
7. In the PR, show the changes instead of listing them: simple before and
   after screenshots with arrows and a few words per change. Add each round's
   scores, and put the final screenshots in the chat too.
