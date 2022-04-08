# Modals

You can use the `Modals` server to pop up the following objects:

- Notifications
  - Static: shows a message plus an "Okay" button
  - Dynamic: can sequence through multiple messages
- Checkboxes (multi-select list)
- Radioboxes (single-select list)
- Text Entry with validator
- Progress bars

To use `Modals` in your code, you will need to add `modals` to your `Cargo.toml` file. From an application in the `apps` directory:

```toml
modals = {path = "../../services/modals"}
```

In all of the examples, you will need this pre-amble to create the `modals` object. The object can be re-used as many times as you like.

```rust,noplayground,ignore
// connect to the modals object through the name resolver
let xns = XousNames::new().unwrap();
let modals = modals::Modals::new(&xns).unwrap();
```

## Static Notification

```rust,noplayground,ignore
modals.show_notification("This is a test!").expect("notification failed");
```

This will pop up a notification that says "This is a test!". Execution blocks at this line until the user pressses any key to acknowledge the notification.

## Progress bar

One can create a progress bar using the `start_progress()` method, with the following parameters:
- `name`: A `&str` that is the title of the progress bar
- `start`: A `u32` that is the starting ordinal
- `end`: A `u32` that is the ending ordinal
- `current`: A `u32` that is the initial point of the progress bar

`start` should be less than `end`, and `current` should be between `start` and `end`, inclusive.

Once the bar is created, you can update its progress using the `update_progress()` method. It takes a number that represents the current progress between the start and end ordinal.

The progress bar is closed by calling the `finish_progress()` method.

```rust,noplayground,ignore
// the ticktimer is used just to introduce a delay in this example. Normally, you'd do something computationally useful instead of just waiting.
let tt = ticktimer_server::Ticktimer::new().unwrap();

let start = 1;
let end = 20;
modals.start_progress("Progress Quest", start, end, start).expect("couldn't raise progress bar");

for i in (start..end).step_by(2) {
    modals.update_progress(i).expect("couldn't update progress bar");
    tt.sleep_ms(100).unwrap();
}
modals.finish_progress().expect("couldn't dismiss progress bar");
```


## Dynamic Notifications
Dynamic notifications are notifications which don't have an option for the user
to close them; instead, the calling program controls when the dialog can
be closed, and can also dynamically update the message. This is useful for
displaying, for example, multi-phase progress updates without stopping and
waiting for a user to hit "OK".

The API is similar to that of the Progress Bar, in that there are start, update,
and close phases:

- To pop up the dynamic notification, use the `dynamic_notification(title: Option<&str>, text: Option<&str>)`
method. The both `title` and `text` are optional, but at least one is recommended, otherwise
you get an empty notification.
- Updates to the notification are done using `dynamic_notification_update(title: Option<&str>, text: Option<&str>)`.
Arguments that are `None` do not update, and show the same text as before.
- Once you are finished showing the set of notifications, you must close the
dialog with `dynamic_notification_close()`.


## Text entry
One can request text entry using the `get_text()` method. This takes the following parameters:

- `prompt`: A `&str` that is the prompt to the user
- `validator`: An `Option<fn(TextEntryPayload, u32) -> Option<ValidatorErr>>`. This is an optional function that takes the text entry payload, along with a dispatch opcode. The dispatch opcode allows a single validator function to be re-used across multiple invocations of `get_text()`.
- `validator_op`: An `Option<u32>`. When `Some()`, the argument inside is passed to the validator to indicate which type of text is being validated.

The idea behind the `validator_op` is that you could create an `Enum` type that specifies the type of text you're entering, and you would pass the `u32` version of that `Enum` to the `get_text()` call so that a single `validator` function can be used to check multiple types of text entry.

```rust,noplayground,ignore
// you can also use the num_derive crate to have bi-directional transformation of the enum
enum ValidatorOp {
    Int2 = 0,
    Int = 1,
}
//
fn my_code() {
    // ... insert code to create modals object, etc.
    match modals.get_text("Input an integer greater than 2", Some(test_validator), Some(ValidatorOp::Int2 as u32)) {
        Ok(text) => {
            log::info!("Input: {}", text.0);
        }
        _ => {
            log::error!("get_text failed");
        }
    }
    match modals.get_text("Input any integer", Some(test_validator), Some(ValidatorOp::Int2 as u32)) {
        Ok(text) => {
            log::info!("Input: {}", text.0);
        }
        _ => {
            log::error!("get_text failed");
        }
    }
}

fn test_validator(input: TextEntryPayload, opcode: u32) -> Option<xous_ipc::String::<256>> {
    let text_str = input.as_str();
    match text_str.parse::<u32>() {
        Ok(input_int) =>
        if opcode == ValidatorOp::Int2 as u32 {
            if input_int <= 2 {
                return Some(xous_ipc::String::<256>::from_str("input must be larger than 2"))
            } else {
                return None
            }
        } else if opcode == ValidatorOp::Int as u32 {
            return None
        } else {
            panic!("unknown discriminant");
        }
        _ => return Some(xous_ipc::String::<256>::from_str("enter an integer value"))
    }
}
```

## Radio Box
A radio box is a mechanism to force a user to pick exactly one item from a list of options.

One can construct a radio box by first repeatedly calling `add_list_item()` with a `&str`
description of the items to select, and then calling `get_radiobutton()` with a `&str` of
the prompt. The returned value will be the `&str` description of the selected item.

Note that upon completion of the radio box, the list of items is automatically cleared
in preparation for another invocation of `modals`.

```rust,noplayground,ignore
const RADIO_TEST: [&'static str; 4] = [
    "zebra",
    "cow",
    "horse",
    "cat",
];

for item in RADIO_TEST {
    modals.add_list_item(item).expect("couldn't build radio item list");
}
match modals.get_radiobutton("Pick an animal") {
    Ok(animal) => log::info!("{} was picked", animal),
    _ => log::error!("get_radiobutton failed"),
}
```

## Check Box
A check box is a mechanism to present a user with a list of several options, of which
they can select none, some, or all of them.

The usage is nearly identical to the Radio Box above, except that the return value
is a `Vec::<String>`. The `Vec` will be empty if no elements are selected.

```rust,noplayground,ignore
const CHECKBOX_TEST: [&'static str; 5] = [
    "happy",
    "😃",
    "安",
    "peaceful",
    "...something else!",
];

for item in CHECKBOX_TEST {
    modals.add_list_item(item).expect("couldn't build checkbox list");
}
match modals.get_checkbox("You can have it all:") {
    Ok(things) => {
        log::info!("The user picked {} things:", things.len());
        for thing in things {
            log::info!("{}", thing);
        }
    },
    _ => log::error!("get_checkbox failed"),
}
```