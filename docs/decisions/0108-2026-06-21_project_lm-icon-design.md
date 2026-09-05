# ADR 0108 - LM Brand Icon: a Black-and-White Split Heart

## Status

Implemented

## Date

2026-06-21

## Background

The original favicon (a heart outline layered with an ECG pulse line) had its two paths interfering with each other at 16/32px sizes, making it overly complex and hard to recognize. The design goal: keep "heart" as the core identifying symbol for the life theme, while echoing Life Matters's theme of being pulled between internal and external forces, since life is constrained simultaneously by internal (medical/physiological) and external (sociological) forces, and this internal-external relationship is itself unstable.

## Decision

- Graphic: a single heart outline is split in two along its vertical centerline, with the foreground (the heart) and background colors inverted between black and white on each half, black background with a white heart on the left, white background with a black heart on the right. The color relationship between "inner" (the heart) and "outer" (the background) swaps between the two sides, with no arrows, boxes, or other extra shapes introduced, using only one shape vocabulary (the heart) to express opposition and interpenetration.
- Heart proportions and position: the heart is shrunk to 85% of its original size; its vertical position is not centered on the geometric bounding box, but computed so that the distance from the heart's two lobes' actual highest points (the arc's apex, not the dip in the middle) and from its bottom tip to the icon's top and bottom edges is equal.
- Icon ownership: the black-and-white version is bound to the LM model / LM format layer (the specification/theory layer), not to any specific piece of software. The LM Simulator (the sim_gui engine in the `life-matters` repository) uses brand green as its primary color per the existing UI conventions, so the same-shaped icon in that repository uses a green version instead (black replaced with `#007A33`, white unchanged). Both the browser favicon and the top-bar logo component use this fixed color value regardless of the app's light or dark mode, since the icon itself represents a fixed brand identity that should not follow the UI theme's light/dark switching.
- Border: three treatments were compared (no border, a uniform black outline all the way around, and a left/right split-color outline), and all three versions are kept in the internal design record for future reference. No border was ultimately chosen, since the black-and-white swap already expresses the internal-external relationship completely, and a border would be an extra graphic element unrelated to the theme, making it feel out of place instead. Deployed to this repository's `icon.svg` and the sim repository's `sim_gui/public/favicon.svg` and top-bar logo, with no version number in the filename.
