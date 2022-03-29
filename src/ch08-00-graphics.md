# Graphics Toolkit

The Xous UX stack consists of three levels:

1. `Modals` and `Menu`s
2. The `GAM` (Graphical Abstraction Manager)
3. The `graphics-server`

# Overview
## Modals and Menus
The `Modals` and `Menu` objects are pre-defined primitives that simplify the creation of Notifications, Checkboxes, Radioboxes, Text Entry boxes, Progress Bars, and Menus. They are as close as you get to a graphics toolkit in Xous.

## GAM
Xous has a security-aware UX infrastructure that aims to make it difficult for rogue processes to pop up dialog boxes that could visually mimic system messages and password boxes.

The `GAM` is a layer that intermediates between the graphics toolkit and the hardware drivers, and enforces these security policies. It does this through the `Canvas` and `Layout` primitives.

The `Canvas` enforces a particular trust level associated with a region of the screen. White text on a black background is reserved for secure, trusted messages, and the `GAM` in combination with the trust level encoded in a `Canvas` is responsible for enforcing that rule. This is also where the `deface` operation occurs, the series of random lines that appear on items in the background.

`Layout`s contain one or more `Canvas` objects and are used to define, at a coarse level, regions of the screen, such as where the status bar belongs, the IME, and so forth.

## Graphics Server
The `graphics-server` is responsible for rendering primitives such as circles, lines, and glyphs to the frame buffer. It places no restrictions on where pixels may be placed.

The `graphics-server` uses the `xous-names` registry mechanism to restrict its access. No user processes can talk directly to it as a result.

