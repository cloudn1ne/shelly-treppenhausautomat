# shelly-treppenhausautomat
Shelly 2 PRO Treppenhausautomat

## Input settings
Make sure to set your Input/Output settings to "Button", "Detached" - Action onpower on for XXX "Turn OFF"

## Compatibility
Tested on a Shelly Pro2 Firmware 1.5.1

```
let RELAY_ID = 0;
let INPUT_ID = 0;

let TIMER_DURATION_MS = 60000;      // Total on time
let BLINK_BEFORE_OFF_MS = 10000;     // Start blinking 10s before turn-off
let BLINK_INTERVAL_MS = 500;         // Blink every 0.5s
let BLINK_COUNT = 3;                 // 3 blinks (on/off cycles)

let off_timer = null;
let blink_timer = null;
let blink_step = 0;
let original_state = true;

// Turn off relay and clear timers
function turnOffRelay() {
  print("Timer expired → turning off relay");
  Shelly.call("Switch.Set", { id: RELAY_ID, on: false });
  clearAllTimers();
}

// Clear all timers
function clearAllTimers() {
  if (off_timer !== null) {
    Timer.clear(off_timer);
    off_timer = null;
  }
  if (blink_timer !== null) {
    Timer.clear(blink_timer);
    blink_timer = null;
  }
  blink_step = 0;
}

// Start blink sequence (3 blinks)
function startBlinkSequence() {
  print("Starting blink sequence");

  blink_step = 0;
  original_state = true;

  blink_timer = Timer.set(BLINK_INTERVAL_MS, true, function () {
    blink_step += 1;

    if (blink_step > BLINK_COUNT * 2) {
      // End blink sequence, ensure light stays ON
      print("Blinking done");
      Timer.clear(blink_timer);
      blink_timer = null;
      Shelly.call("Switch.Set", { id: RELAY_ID, on: true });
      return;
    }

    // Toggle relay
    original_state = !original_state;
    Shelly.call("Switch.Set", { id: RELAY_ID, on: original_state });
  });
}

// Event handler
Shelly.addEventHandler(function (event) {
  print("Event: " + JSON.stringify(event));

  if (event.info.component === "input:" + INPUT_ID && event.info.event === "btn_down") {
    Shelly.call("Switch.GetStatus", { id: RELAY_ID }, function (status) {
      if (status.output || blink_timer !== null) {
        // Turn off and cancel
        print("Manual cancel → turning off");
        Shelly.call("Switch.Set", { id: RELAY_ID, on: false });
        clearAllTimers();
      } else {
        // Turn on and start timers
        print("Turning on relay and scheduling timers");
        Shelly.call("Switch.Set", { id: RELAY_ID, on: true });

        // Schedule blink warning
        Timer.set(TIMER_DURATION_MS - BLINK_BEFORE_OFF_MS, false, startBlinkSequence);

        // Schedule auto-off
        off_timer = Timer.set(TIMER_DURATION_MS, false, turnOffRelay);
      }
    });
  }
});
```
