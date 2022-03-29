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

```Rust
// connect to the modals object through the name resolver
let xns = XousNames::new().unwrap();
let modals = modals::Modals::new(&xns).unwrap();
```

## Static Notification

```Rust
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

```Rust
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

## Text entry
