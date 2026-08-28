# Smart Portable Medication Reminder and Tracking Box

## 1. Project Overview

This project is a **compact, portable smart medication reminder and
medicine-interaction tracker**. The main purpose is to help users
remember scheduled medicines and reduce confusion when multiple
medicines must be taken at different times.

The device **does not automatically dispense or physically move
tablets**. The user stores medicine strips in dedicated compartments and
removes the required tablets manually.

The Version 1 system focuses on:

-   Timed medicine reminders
-   Detecting which medicine compartments were opened and closed
-   Warning when a non-required compartment is opened
-   Asking the user for final confirmation
-   Saving the current medication session so it can recover after power
    loss
-   Running from a rechargeable battery
-   Remaining compact and portable

------------------------------------------------------------------------

## 2. Physical Structure

The enclosure is designed as a **4 × 2 grid**, giving a total of **8
equal front-facing sections**.

  Section   Purpose
  --------- -------------------------------------
  1         Touch display
  2--8      Seven medicine storage compartments

The seven medicine compartments represent all possible combinations of
three daily sessions:

-   Morning
-   Afternoon
-   Night

### 2.1 Compartment Combination System

  Compartment   Schedule Combination
  ------------- -----------------------------
  C1            Morning + Afternoon + Night
  C2            Morning + Afternoon
  C3            Morning + Night
  C4            Afternoon + Night
  C5            Morning only
  C6            Afternoon only
  C7            Night only

This organization avoids duplicating the same medicine strip in multiple
time-based boxes.

### Example

If a user has:

-   Medicine A: Morning + Afternoon + Night → C1
-   Medicine B: Morning + Afternoon → C2
-   Medicine C: Morning only → C5
-   Medicine D: Night only → C7

Then at **morning time**, the device indicates the compartments
containing morning medicines:

-   C1
-   C2
-   C3
-   C5

At **afternoon time**:

-   C1
-   C2
-   C4
-   C6

At **night time**:

-   C1
-   C3
-   C4
-   C7

------------------------------------------------------------------------

## 3. Main Working Principle

The device follows this basic sequence:

1.  User configures medicine schedules.
2.  ESP32 stores the schedule and obtains the current time.
3.  The device waits for the next medication time.
4.  At the scheduled time:
    -   Buzzer activates.
    -   Display shows the active medication session.
    -   Required compartments are highlighted.
5.  User opens the required compartment(s).
6.  Microswitches detect lid movement.
7.  If a wrong compartment is opened:
    -   A warning sound is produced.
    -   The display shows that the compartment is not required.
8.  The user takes the required medicine.
9.  User closes the compartment lids.
10. Once all required compartments have been opened and closed, the
    display asks:

**"Have you taken all your medicines?"**

11. User selects:

-   **YES** → session is marked complete and the next timer is
    scheduled.
-   **NO** → session remains active and the user can continue or retry.

12. The important session state is saved in non-volatile memory.
13. If power is lost and later restored, the device resumes from the
    last saved state.

------------------------------------------------------------------------

# 4. System Block Diagram

``` text
                     ┌──────────────────────┐
                     │     Touch Display    │
                     │   Status + YES/NO    │
                     └──────────▲───────────┘
                                │
┌──────────────┐                │
│  7 Lid       │                │
│ Microswitches├───────┐        │
└──────────────┘       │        │
                       ▼        ▼
                  ┌─────────────────┐
                  │      ESP32      │
                  │ Main Controller │
                  │ Wi-Fi + Storage │
                  └───────┬─────────┘
                          │
                 ┌────────┴─────────┐
                 ▼                  ▼
          ┌─────────────┐     ┌─────────────┐
          │   Buzzer    │     │ NVS Memory  │
          │  Reminder   │     │ Dose State  │
          └─────────────┘     └─────────────┘
                          ▲
                          │
                  ┌───────┴────────┐
                  │ Power System   │
                  │ Battery        │
                  │ Charging       │
                  │ Regulation     │
                  └────────────────┘
```

------------------------------------------------------------------------

# 5. Workflow Flowchart

``` text
┌──────────────┐
│ Power ON     │
└──────┬───────┘
       ▼
┌─────────────────────────┐
│ Load saved device state │
└──────┬──────────────────┘
       ▼
┌─────────────────────────┐
│ Get current time        │
│ Wi-Fi / NTP if needed   │
└──────┬──────────────────┘
       ▼
┌─────────────────────────┐
│ Check next medicine time│
└──────┬──────────────────┘
       ▼
    Is it time?
       │
   No ─┘
       │ Yes
       ▼
┌─────────────────────────┐
│ Start medication session│
│ Activate buzzer         │
│ Show required boxes     │
└──────┬──────────────────┘
       ▼
┌─────────────────────────┐
│ Monitor all lid switches│
└──────┬──────────────────┘
       ▼
   Lid opened?
       │
       ▼
 Is lid required?
   │            │
  Yes           No
   │             │
   ▼             ▼
Mark required   Warning beep
box as opened   Show wrong box
   │
   ▼
Wait for lid to close
   │
   ▼
Mark box interaction complete
   │
   ▼
All required boxes completed?
   │
 No ────────────────┐
                    │
                    ▼
                Continue
                    │
                   Yes
                    ▼
┌──────────────────────────┐
│ Ask: Taken all medicine? │
│       YES / NO           │
└──────┬─────────────┬─────┘
       │ YES         │ NO
       ▼             ▼
Save session      Keep session
as completed      active / retry
       │
       ▼
┌─────────────────────────┐
│ Calculate next schedule │
└──────┬──────────────────┘
       ▼
      WAIT
```

------------------------------------------------------------------------

# 6. Core Logic

The system should be implemented as a **state machine** rather than a
large collection of unrelated conditions.

## 6.1 Recommended States

### 1. IDLE

No medication session is currently active.

The ESP32: - Displays time/status - Checks the next medication
schedule - Uses low power where possible

### 2. REMINDER_ACTIVE

The scheduled medication time has arrived.

The system: - Activates the buzzer - Displays the session - Shows
required compartments

### 3. TRACKING_COMPARTMENTS

The system monitors the seven lid switches.

It records: - Which compartment opened - Whether it was required -
Whether it was closed again

### 4. AWAITING_CONFIRMATION

All required compartment interactions have been completed.

The display asks:

**Have you taken all your medicines?**

### 5. COMPLETED

The user presses **YES**.

The ESP32: - Saves completion status - Saves timestamp - Stops
reminder - Moves to the next schedule

### 6. RETRY / NOT_CONFIRMED

The user presses **NO**.

The session remains active. The system can: - Show instructions again -
Restart a reminder after a delay - Allow the user to repeat the process

### 7. RECOVERY

Used after unexpected power loss.

The ESP32: - Reads saved state from non-volatile memory - Determines
whether a medication session was active - Restores the correct screen
and workflow

------------------------------------------------------------------------

# 7. Switch Logic

Each medicine compartment has one microswitch.

A possible interpretation:

  Lid Condition   Switch State
  --------------- --------------
  Lid closed      PRESSED
  Lid open        RELEASED

The exact HIGH/LOW logic depends on whether the switch is wired using
pull-up or pull-down configuration.

## Recommended Wiring Logic

Use:

-   ESP32 internal pull-up resistor
-   Switch connected to GND
-   Normally closed or normally open based on mechanical design

The firmware should use **debouncing** to avoid false triggers caused by
mechanical contact bounce.

### Important Software Events

For each compartment:

``` text
CLOSED → OPEN
OPEN → CLOSED
```

A valid medicine interaction can be:

``` text
Initial closed state
        ↓
Required compartment opens
        ↓
System records "opened"
        ↓
Compartment closes
        ↓
System records "interaction completed"
```

------------------------------------------------------------------------

# 8. Example Morning Session Logic

Assume the morning requires:

-   C1: Morning + Afternoon + Night
-   C2: Morning + Afternoon
-   C3: Morning + Night
-   C5: Morning only

The required mask can conceptually be:

``` text
Required:
C1 = TRUE
C2 = TRUE
C3 = TRUE
C4 = FALSE
C5 = TRUE
C6 = FALSE
C7 = FALSE
```

During the session:

### User opens C1

-   Correct compartment
-   Record C1 opened
-   User closes C1
-   Record C1 completed

### User opens C6

-   Wrong compartment
-   Warning beep
-   Display warning

### User completes C1, C2, C3 and C5

-   Display asks for confirmation

### User presses YES

-   Morning session completed
-   Save status
-   Set next timer

------------------------------------------------------------------------

# 9. Main Components Required

## 9.1 Main Controller

### ESP32 Development Board

Purpose: - Main processing - Wi-Fi connectivity - Reading switches -
Controlling display - Controlling buzzer - Managing schedules - Saving
state

A development board can be used for the first prototype. A smaller
custom ESP32 board can be designed later for compactness.

------------------------------------------------------------------------

## 9.2 Touch Display

Possible options:

-   Small TFT touch display
-   ESP32-compatible touch display
-   Compact smart display module

Functions:

-   Show current time
-   Show current medication session
-   Indicate required compartments
-   Show warnings
-   Provide YES / NO confirmation
-   Provide basic settings interface

The display occupies one of the eight front sections.

------------------------------------------------------------------------

## 9.3 Seven Microswitches

One switch per medicine lid.

Total:

``` text
7 compartments = 7 microswitches
```

Purpose:

-   Detect lid open/closed status
-   Detect interaction with the correct compartment
-   Detect opening of an incorrect compartment

Recommended considerations:

-   Small size
-   Long mechanical life
-   Reliable actuator
-   Easy mounting in the 3D printed enclosure

------------------------------------------------------------------------

## 9.4 Buzzer

A small active buzzer is recommended for Version 1.

Purpose:

-   Medication reminder
-   Wrong compartment warning
-   User feedback

Different beep patterns can be used:

``` text
Normal reminder:  Beep ... Beep ... Beep
Wrong box:        Beep-Beep-Beep
Session complete: Long confirmation beep
```

------------------------------------------------------------------------

## 9.5 Rechargeable Battery

Recommended general direction:

-   Single-cell rechargeable lithium battery
-   Capacity selected after measuring actual power consumption
-   Flat battery shape for compact enclosure design

Battery capacity should be chosen based on:

-   ESP32 power consumption
-   Display consumption
-   Wi-Fi usage
-   Buzzer usage
-   Desired operating time

------------------------------------------------------------------------

## 9.6 Battery Charging Module

Required functions:

-   USB charging input
-   Lithium battery charging
-   Battery protection
-   Charge indication if required

USB-C is recommended for physical convenience.

------------------------------------------------------------------------

## 9.7 Power Management

This is an important part of the project.

The design may require:

``` text
USB-C
  │
  ▼
Charging Circuit
  │
  ▼
Rechargeable Battery
  │
  ▼
Power Regulation
  │
  ▼
ESP32 + Display + Switches + Buzzer
```

Important considerations:

-   Correct voltage for ESP32
-   Correct voltage/current for display
-   Battery protection
-   Deep-discharge protection
-   Low-battery detection

------------------------------------------------------------------------

## 9.8 3D Printed Enclosure

The enclosure should include:

-   4 × 2 front layout
-   Seven medicine compartments
-   One display compartment
-   Individual lids
-   Microswitch mounting points
-   Electronics layer
-   Battery space
-   Charging port opening
-   Buzzer opening
-   Screws or snap-fit features
-   Optional removable/cleanable trays

The electronics should ideally be separated from the medicine storage
area.

------------------------------------------------------------------------

# 10. Additional Components You Should Not Forget

These are easy to overlook.

## RTC / Time Backup

If Wi-Fi is unavailable after restart, the device still needs accurate
time.

Possible approaches:

### Option A: NTP + ESP32 time

Simple, but depends on having time available after power recovery.

### Option B: External RTC

An RTC module with backup battery can maintain time even when the main
battery is fully discharged.

For a reliable medication device, an external RTC is worth considering.

------------------------------------------------------------------------

## Non-Volatile Memory

The ESP32 has internal non-volatile storage options.

Store important information such as:

-   Current session
-   Next schedule
-   Completion state
-   Required compartment mask
-   Last completed dose
-   Configuration settings

Do **not** write continuously on every loop. Save only when meaningful
events occur.

------------------------------------------------------------------------

## Pull-Up/Pull-Down and Debouncing

Seven mechanical switches can generate noisy signals.

Required in software:

-   Debounce delay or state-based debounce
-   Stable transition detection

------------------------------------------------------------------------

## Hinges and Mechanical Tolerances

The lid must reliably activate the microswitch.

Test:

-   Lid closed normally
-   Lid slightly misaligned
-   Repeated opening and closing
-   3D print tolerance changes

The physical mechanism should be tested before finalizing the
electronics board.

------------------------------------------------------------------------

## Low Battery Warning

The display should warn:

``` text
LOW BATTERY
Please charge the device
```

Optional future behavior:

-   Low battery icon
-   Warning buzzer
-   Automatic Wi-Fi reduction

------------------------------------------------------------------------

# 11. Suggested Firmware Data Structure

Conceptually:

``` text
Medicine Schedule

Morning:
[C1, C2, C3, C5]

Afternoon:
[C1, C2, C4, C6]

Night:
[C1, C3, C4, C7]
```

For each active session:

``` text
Session Active = TRUE

Required:
C1 = TRUE
C2 = TRUE
C3 = TRUE
C4 = FALSE
C5 = TRUE
C6 = FALSE
C7 = FALSE

Opened:
C1 = FALSE
C2 = FALSE
...

Completed:
C1 = FALSE
C2 = FALSE
...
```

After interaction:

``` text
If all required compartments are completed:
    Show YES / NO confirmation
```

------------------------------------------------------------------------

# 12. Power-Loss Recovery Logic

When an important event occurs, save the state.

Examples:

``` text
Session started
Required compartment opened
Required compartment closed
User pressed YES
Next timer calculated
```

After reboot:

``` text
Power ON
   ↓
Read saved state
   ↓
Was a session active?
   ├── NO → Normal IDLE mode
   │
   └── YES → Restore active session
                 ↓
             Continue tracking
```

This means the device does not forget the medication session if power is
unexpectedly lost.

------------------------------------------------------------------------

# 13. Suggested Development Plan

## Phase 1: Basic Electronics

Build:

-   ESP32
-   One switch
-   One buzzer
-   Simple display

Test:

-   Scheduled reminder
-   Switch detection
-   Touch confirmation

------------------------------------------------------------------------

## Phase 2: Seven Compartment Simulation

Connect:

-   Seven switches
-   Display
-   Buzzer

Test:

-   Correct compartment
-   Wrong compartment
-   Multiple required compartments
-   Open and close sequence

------------------------------------------------------------------------

## Phase 3: State Persistence

Add:

-   Non-volatile storage
-   Save/restore state

Test:

-   Start medication session
-   Disconnect power
-   Restart
-   Confirm correct recovery

------------------------------------------------------------------------

## Phase 4: Battery System

Add:

-   Rechargeable battery
-   Charging circuit
-   Battery voltage monitoring

Test:

-   Normal operation
-   Charging
-   Low battery
-   Unexpected shutdown

------------------------------------------------------------------------

## Phase 5: 3D Printed Prototype

Create:

-   4 × 2 enclosure
-   Seven lids
-   Switch mounts
-   Display opening
-   Electronics compartment
-   Battery compartment

------------------------------------------------------------------------

# 14. Important Project Limitations

The device can confirm:

-   The correct compartment was opened
-   The compartment was closed
-   The user manually confirmed taking medicine

The device **cannot guarantee that the user actually swallowed the
medicine**.

Therefore, the correct project description is:

> **A smart medication reminder and medicine-compartment interaction
> tracking system.**

It should not claim to medically guarantee medicine ingestion.

------------------------------------------------------------------------

# 15. Version 1 Final Architecture

``` text
        ┌─────────────────────────────┐
        │   4 × 2 SMART MEDICINE BOX  │
        │                             │
        │ [C1] [C2] [C3] [DISPLAY]    │
        │ [C4] [C5] [C6] [C7]         │
        └──────────────┬──────────────┘
                       │
                 7 Microswitches
                       │
                       ▼
                  ┌─────────┐
                  │  ESP32  │
                  └────┬────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
     Touch UI       Buzzer       Storage
                                   │
                                  NVS
                       │
                       ▼
                 Power System
                       │
        USB-C → Charger → Battery
```

------------------------------------------------------------------------

# 16. Final Project Summary

The proposed device is a **portable 4 × 2 smart medication box** with:

-   8 equal front sections
-   1 touch display section
-   7 medicine compartments
-   7 microswitches for lid interaction tracking
-   ESP32 controller
-   Buzzer reminders
-   Wi-Fi-based configuration
-   Time scheduling
-   YES/NO medication confirmation
-   Wrong-compartment warnings
-   Non-volatile state storage
-   Rechargeable battery
-   USB charging
-   3D printed enclosure

The key design principle is:

> **Organize medicine by its combination of required daily sessions
> rather than forcing the same medicine to be duplicated across Morning,
> Afternoon, and Night boxes.**

This keeps the device compact while supporting all possible combinations
of a three-session daily medication schedule.
