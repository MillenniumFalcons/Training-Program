# Step 8: Coding a Full Robot

You've built a minibot drivetrain, added sensors, swerve, and TalonFX subsystems. Now you're coding an actual competition robot: **Droopy McCool, Team 3647's 2024 robot**.

*Timeline: as long as it takes. Ask for help when stuck.*

> Note: this step is still in progress.

---

## The Game: 2024 CRESCENDO

Search up the game reveal video on youtube, something like "FRC 2024 Crescendo Game Reveal".

The game piece is a foam ring called a **Note**. The robot:
1. **Intakes** a Note off the ground with the intake rollers + wrist
2. **Transfers** it through a handoff into the **Kicker** (inside the robot)
3. **Pivots** to the correct shooting angle
4. **Shoots** by spinning up the shooter wheels and kicking the Note through

Ask a lead to walk you through the physical robot before writing any code.

---

## Subsystems

Droopy McCool has these subsystems:

| Subsystem | Type | Purpose |
|-----------|------|---------|
| `SwerveDrive` | (done in Step 6) | Drives the robot |
| `Intake` | (done in Step 7) | Roller that grabs notes off the ground |
| `Wrist` | (done in Step 7) | Arm that deploys the intake |
| `Kicker` | TalonFXSubsystem | Feeds the note into the shooter |
| `LeftShooter` | TalonFXSubsystem | Left flywheel |
| `RightShooter` | TalonFXSubsystem | Right flywheel |
| `Pivot` | TalonFXSubsystem | Tilts the shooter to aim |

Find CAN IDs in Phoenix Tuner.

---

## New Concepts You Need

### Follower Motors

Some mechanisms have two motors driving the same shaft. One motor is the **master** — it receives the control request. The other is a **slave** (follower) — it mirrors the master automatically.

In `TalonFXSubsystem`, call `addFollower(slaveTalonFX, alignmentValue)` in your subsystem constructor after `super(...)`.

`MotorAlignmentValue` controls whether the follower spins the same direction as the master or opposite:
- `Aligned` — same direction (e.g., Pivot: both motors push the same way)
- `Opposed` — opposite direction (e.g., Shooters: motors face each other, so one must be inverted)

```java
// in your constructor, after super(...)
addFollower(slave, MotorAlignmentValue.Aligned);
```

The slave motor still needs its own constants and configuration applied in your constants file.

---

### The Superstructure

As the robot gets more complex, you need **coordinated multi-subsystem behaviors**: intaking requires the wrist to go down AND the intake to spin at the same time. Handoff requires stopping the intake, moving wrist and pivot to a specific angle, then running the kicker until the note is detected.

The **Superstructure** is a plain Java class (not a `Subsystem`) that holds references to all subsystem command classes and exposes high-level robot behaviors as `Command`s. `RobotContainer` binds buttons to Superstructure commands instead of directly to subsystem commands.

```
RobotContainer → Superstructure.intake() → parallel(wristCommands.down(), intakeCommands.intake())
```

The Superstructure also tracks **robot state** — whether it's intaking, has a note, etc. — using a `SuperstructureState` enum.

---

### Current-Based Triggers

You don't need a separate sensor to detect when the intake grabs a Note. When the rollers suddenly have extra load, motor **current draw spikes**. You can create a WPILib `Trigger` on this:

```java
Trigger intakeCurrentTrigger = new Trigger(() -> intake.getMasterCurrent() > 30).debounce(0.1);
```

`.debounce(0.1)` means the trigger must be true for 100 ms continuously before it fires — this filters out the spin-up current spike.

Use these triggers in `RobotContainer` to automatically start handoff when a Note is detected, rather than requiring the driver to press a button.

---

### AutoDrive (Auto-Aim)

For aiming at the speaker, the robot needs to:
1. **Rotate** to face the speaker — calculate the angle from the robot's odometry pose to the speaker pose, then use a PID controller on heading error
2. **Set pivot angle** based on distance — use `Math.atan(height / distance)` to calculate the needed angle

`AutoDrive` is a utility class (not a subsystem) that wraps these calculations and exposes them as commands. It gets the robot's current pose from a `Supplier<Pose2d>`.

The speaker's field position changes based on alliance color. Use `AllianceChecker` to update the target pose when alliance is determined.

---

## What to Build

Work through these in order. Each milestone should be tested on the robot before moving to the next.

### Milestone 1: Make Everything Move

Build the remaining subsystems and their command classes, one at a time:

**Kicker** — extends `TalonFXSubsystem`. Open-loop only (same pattern as Intake). Also holds two Time-of-Flight sensors and exposes `noteInKicker()` and `noteReadyToShoot()` boolean methods based on sensor range. Ask a lead about the ToF library and how to read from the sensors.

**LeftShooter / RightShooter** — each extends `TalonFXSubsystem`. Open-loop only. Each has a slave motor — use `addFollower` with `Opposed` alignment (the motors face each other on the shooter).

**Pivot** — extends `TalonFXSubsystem` and implements the `Pivot` interface. Like `Wrist`, but with a slave motor (`Aligned`). Has `setAngle`, `angleReached`, `setEncoder`, and `getPosition`. Also expose `calculateAngle(double triggerValue)` which linearly maps trigger input (0–1) to an angle between min and max.

Create constants files for each. Use the CAN IDs from `GlobalConstants`. Check the CAD for gear ratios and physical limits.

Add everything to `RobotContainer` and bind simple test commands to controller buttons. Confirm every mechanism physically moves before going further.

---

### Milestone 2: Intake & Handoff Logic

Build the **Superstructure** class. It takes all subsystems and all command classes as constructor arguments.

Implement these Superstructure commands:

**`intake()`** — deploys the wrist and spins the intake at the same time (parallel).

**`endIntake()`** — stows the wrist, stops the intake.

**`transferNote(Trigger kickerCurrentTrigger)`** — the full handoff sequence:
1. Stop intake
2. Move pivot and wrist to the handoff angle (parallel, wait for both)
3. Run intake + kicker together
4. Reverse-kick slightly to seat the note

Add current triggers to `RobotContainer`. Wire up:
- Right bumper held → `intake()`; released → `endIntake()`
- `intakeCurrentTrigger` fires → automatically start `transferNote`

---

### Milestone 3: Shooting

Implement shooting in the Superstructure:

**`spinShooters(DoubleSupplier triggerValue)`** — spin both shooter wheels based on trigger. Right and left wheels run at different speeds (tune this).

**`kickShoot()`** — wait a short time for shooters to spin up, then run the kicker to feed the note through.

**`stopShooters()`** — stop everything.

Wire up in `RobotContainer`: right trigger held → spin shooters, fire `kickShoot` on press.

For fixed-angle shots, bind controller buttons to `pivotCommands.setAngle(angle)` for a few preset angles (e.g., close shot, mid, far). Tune these angles on the robot.

---

### Milestone 4: Auto-Aim

Build the **`AutoDrive`** utility class. It needs:
- A `Supplier<Pose2d>` (from `drivetrain::getOdoPose`)
- A `Superstructure` reference

Implement:
- `getSpeakerRotVel()` — use a `PIDController` to generate a rotation rate that points the robot at the speaker
- `calculatePivotAngleToSpeaker()` — `Math.atan(heightDifference / distance)`, converted to degrees
- `setPivotAngleToSpeaker()` — wraps the above as a `pivotCommands.setAngle(supplier)` command

In `DrivetrainCommands.drive(...)`, check `autoDrive.getDriveMode()` — if it's `SPEAKER`, replace the driver's rotation input with `autoDrive.getSpeakerRotVel()`.

Implement `AllianceChecker` integration so the speaker pose flips correctly on Red vs Blue alliance.

Tune the rotation and pivot PID controllers on the robot.

---

### Milestone 5: Full Cycle Test

With a lead present, run through the full cycle repeatedly:

- [ ] Robot intakes a Note cleanly off the ground
- [ ] Handoff triggers automatically and note ends up in Kicker
- [ ] Pivot moves to the correct angle
- [ ] Shooters spin up and note exits cleanly
- [ ] Auto-aim rotates robot to speaker and sets correct pivot angle
- [ ] Shot scores from multiple distances

---

## Checklist Before Calling It Done

- [ ] All subsystem constants files configure motors in a `static {}` block
- [ ] Kicker detects note presence with ToF sensors
- [ ] Shooter slave motors are `Opposed` (they spin in opposite directions on purpose)
- [ ] Pivot slave motor is `Aligned`
- [ ] Superstructure state machine tracks `INTAKING`, `HANDOFF`, `NOTE_PREP`, `SHOOTING`, `NONE`
- [ ] Handoff is automatic (current trigger → command, no driver input needed)
- [ ] Auto-aim rotates robot to speaker and updates pivot angle from distance
- [ ] Full game cycle works reliably in practice

**Check in with a lead at each milestone before moving on.**
