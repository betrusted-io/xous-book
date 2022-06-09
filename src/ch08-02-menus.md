# Menus

Menus are created with the help of the `menu_matic()` convenience call.

Conceptually, a Menu in Xous is a list of `MenuItem`. Graphically, the menus
are rendered in the order that the `MenuItem`s are added to the list. When a `MenuItem` is selected, it fires a message off to another server to effect the corresponding outcome desired of the logical menu description.

Thus, each `MenuItem` has the following fields:

- A `name` describing the menu item, limited to a 64-byte long unicode string
- An `Option` for an `action connection`. This is a CID to a server to which a message will be sent upon selecting the item. If `None`, the menu item does nothing and just closes the menu.
- An `action opcode`. This is a `u32` value that corresponds to the discriminant of the `enum` used to dispatch opcodes in your main loop (e.g., the parameter passed as `msg.body.id()`).
- A `MenuPayload`, which is an `enum` that currently can only be a `Scalar` payload consisting of up to 4 `u32` values. There's a future provision for this to be extended to a small `Memory` message but it is not yet implemented (please [open an issue](https://github.com/betrusted-io/xous-core/issues) if you need this feature, and helpfully remind the maintainers to also update the Xous Book docs once this is done).
- `close on select` - when set to true, the menu will automatically close when the item is selected.

So, when a `MenuItem` is selected by the user, the menu implementation will fire off a `Scalar` message to the server identified by `action_conn` with the opcode specified by `action opcode` and a payload of `MenuPayload`. The receiving server can asynchronously receive this message in its main loop and act upon the menu selection.

The general idiom is to create a `Vec` of `MenuItem`s, and then pass them into `MenuMatic`, as seen below:

```rust,noplayground,ignore
pub fn create_kbd_menu(status_conn: xous::CID, kbd_mgr: xous::SID) -> MenuMatic {
    let mut menu_items = Vec::<MenuItem>::new();

    let code: usize = KeyMap::Qwerty.into();
    menu_items.push(MenuItem {
        name: xous_ipc::String::from_str("QWERTY"),
        action_conn: Some(status_conn),
        action_opcode: StatusOpcode::SetKeyboard.to_u32().unwrap(),
        action_payload: MenuPayload::Scalar([code as u32, 0, 0, 0]),
        close_on_select: true,
    });
    let code: usize = KeyMap::Dvorak.into();
    menu_items.push(MenuItem {
        name: xous_ipc::String::from_str("Dvorak"),
        action_conn: Some(status_conn),
        action_opcode: StatusOpcode::SetKeyboard.to_u32().unwrap(),
        action_payload: MenuPayload::Scalar([code as u32, 0, 0, 0]),
        close_on_select: true,
    });
    menu_items.push(MenuItem {
        name: xous_ipc::String::from_str("Close Menu"),
        action_conn: None,
        action_opcode: 0,
        action_payload: MenuPayload::Scalar([code as u32, 0, 0, 0]),
        close_on_select: true,
    });

    menu_matic(menu_items, gam::KBD_MENU_NAME, Some(kbd_mgr)).expect("couldn't create MenuMatic manager")
}

```

This will create a menu with three items, "QWERTY", "Dvorak", and "Close Menu". When, for example, the "QWERTY" item is selected, it will send a message to the server pointed to be `status_conn`, with the argument of `StatusOpcode::SetKeyboard` as a`u32`, and an argument consisting of `[code, 0, 0, 0,]`. In this case, only `code` has meaning, and the other three are just placeholders.

The third menu item has `None` for the connection, so when it is selected, no messages are sent and the menu is simply closed.

## Raising the Menu

Once you have created your menu, you can cause the menu to pop up with the following `gam` call:

```rust,noplayground,ignore
gam.raise_menu(gam::KBD_MENU_NAME).expect("couldn't raise keyboard layout submenu");
```

The menu will automatically close if `close_on_select` is `true`.

## Permission to Create Menus

What's the `gam::KBD_MENU_NAME` field all about?

In order to prevent rogue processes from creating menus willy-nilly that resemble, for example, the main menu but firing off forged messages to undesired processes, there is an access control list for menus.

The access control list is kept in the `gam`, and can be found in `services/gam/src/lib.rs`. You must add a `const str` that gives your menu a unique name, and insert it into the `EXPECTED_BOOT_CONTEXTS` structure, otherwise, the `gam` will deny the creation of your menu item. The access list is "trust on first use", and secure operations such as accessing the root keys will not be allowed to proceed until all contexts have been allocated. Therefore, if you are creating a menu, you need to call `menu_matic()` early in the boot process, or else you will be unable to unlock the PDDB.

Permissions Checklist:

1. Give your menu a name in `services/gam/src/lib.rs`
2. Add the name to `EXPECTED_BOOT_CONTEXTS`
3. Claim your name with `menu_matic()` early in the boot process

## Modifying the Menu

`menu_matic()` has a third argument, which is an `Option<xous::SID>`. If you never plan to modify your menu, you can leave it as `None`. However, if you want to do things such as dynamically create and remove menu items, or pre-select an index in the menu list, you will need to specify an SID. This is used to create the `MenuMatic` object, which is returned to the caller.

The SID is created as follows:

```rust,noplayground,ignore
let kbd_mgr = xous::create_server().unwrap();
```

`MenuMatic` has the following methods available on it:

- `add_item(MenuItem)` - adds the `MenuItem` specified to the end of the menu list, returning `true` to indicate success.
- `delete_item(&str)` - deletes an item with a `name` specified as the argument. Returns `true` to indicate success.
- `set_index(usize)` - sets the index pointer of the menu to the specified offset. Typically ued to create a "default" position for the menu before it is raised.
- `quit()` - exit and destory the `MenuMatic` server

If you don't need the above functionality, it's recommended that you do not create the server, as it consumes memory and eats up connection and server name space.
