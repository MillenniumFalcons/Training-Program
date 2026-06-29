# Step 7: TalonFX Subsystem

In this step, you will learn how to use TalonFX subsystems in your robot code. You will also learn how to tune the TalonFX controllers using the Phoenix Tuner. This is what you we use to code the actual robot during the season. You will be programming the intake of Droopy McCool, the 2024 robot, which consist of the Intake and Wrist subsystems. The Intake is the rollers that grab the note, and the Wrist is the mechanism that the rollers are attached to which slaps down onto the ground. 

*Timeline ~2 weeks.*

---

## Background: TalonFXSubsystem

Read through [*Control Requests*](https://v6.docs.ctr-electronics.com/en/stable/docs/api-reference/api-usage/control-requests.html) to learn how to run the motors. The TalonFXSubsystem class already handles most of this for you, but you should still know how to do it yourself.

`TalonFXSubsystem` (in `team3647/lib/`) is an abstract base class that wraps one or more TalonFX motors and handles the periodic loop for you:

- `readPeriodicInputs()` — reads position, velocity, current, and voltage from the motor and logs them with AdvantageKit
- `writePeriodicOutputs()` — sends the current `controlMode` (the control request) to the master motor and all followers

Your subsystem extends it and calls protected helpers like `setOpenloop`, `setPositionExpoVoltage`, etc. to set what gets sent.

> **Before writing any code:** read through `TalonFXSubsystem.java` in full. Understand what `positionConversion` and `velocityConversion` do, and why the position helpers divide by `positionConversion` before sending the value to the motor.


---

## Background: Motion Magic

Read through [TalonFX Control Modes](<Supplementals\TalonFX Control Modes.md>) in the Supplementals folder to understand the different control modes.

Motion Magic is CTRE's built-in motion profiling. Instead of jumping straight to a setpoint (which causes a large current spike and jerky motion), it generates a smooth velocity profile and tracks it with a PID loop.

You configure it with:
- **kP, kI, kD** — PID gains for following the profile
- **kS** — static feedforward (voltage to overcome stiction)
- **kV, kA** — velocity and acceleration feedforward
- **kG** - gravity feedforward for mechanisms like pivots and elevators
- **MotionMagicCruiseVelocity** — peak velocity during the move
- **MotionMagicAcceleration** — how fast to ramp up and down
- **MotionMagicExpo_kV, MotionMagicExpo_kA** — used by `MotionMagicExpoVoltage`, which gives smoother low-speed behavior by using an exponential profile instead of a trapezoid. These constants are used to generate the trajectory, not to control the motor. **You should start with high values when tuning these unlike the other constants, because it puts a more restrictive limit on the trajectory.**

The Wrist is position-controlled, so it uses `MotionMagicExpoVoltage`. The Intake doesn't need motion profiling at all — it's just open-loop roller control.

### Tuning in Phoenix Tuner X

Before writing code, test and tune your motor in **Phoenix Tuner X**:
1. Connect to the robot and open the **Devices** tab
2. Select your motor and use the **Control** tab to send open-loop output — confirm the motor moves in the right direction
3. Use the **Config** tab to set your PID and Motion Magic parameters, then test them live with the Control tab

Tune here first, then hard-code the working values into your constants file. **Ask a lead to help you with this**.

---

## Part 1: IntakeConstants

Create `IntakeConstants.java` in your `constants/` folder.

The Intake is open-loop — it just spins rollers, so there's no position control or Motion Magic needed. Your constants file only needs:
- A `TalonFX` instance with the motor's CAN ID
- Nominal voltage, stall current limit, and max current limit
- Motor output config (neutral mode, invert direction)
- Current limit config

Wire everything up in a `static {}` block:

```java
static {
    CurrentLimitsConfigs kMasterCurrent = new CurrentLimitsConfigs();
    MotorOutputConfigs kMasterMotorOutput = new MotorOutputConfigs();
    TalonFXConfigurator kMasterConfigurator = kMaster.getConfigurator();

    kMasterConfigurator.apply(new TalonFXConfiguration());

    kMasterCurrent.StatorCurrentLimitEnable = true;
    kMasterCurrent.StatorCurrentLimit = kMaxCurrent;
    kMasterMotorOutput.NeutralMode = NeutralModeValue.Brake;
    kMasterMotorOutput.Inverted = InvertedValue.CounterClockwise_Positive;

    kMasterConfigurator.apply(kMasterMotorOutput);
    kMasterConfigurator.apply(kMasterCurrent);
}
```

---

## Part 2: Intake Subsystem

Create `Intake.java` in `subsystems/`. It should `extend TalonFXSubsystem`.

The Intake only needs one method beyond what `TalonFXSubsystem` provides:

```java
public void openLoop(double output) {
    super.setOpenloop(output);
}
```

Override `getName()` to return `"Intake"`.

That's it — the base class handles `readPeriodicInputs` and `writePeriodicOutputs` for you.

---

## Part 3: IntakeCommands

Create `IntakeCommands.java` in `commands/`. Take an `Intake` in the constructor.

Implement three commands:

**`intake()`** — spin the rollers inward at a fixed duty cycle:
```java
public Command intake() {
    return Commands.run(() -> intake.openLoop(0.5), intake);
}
```

**`kill()`** — stop the rollers immediately (runOnce so it doesn't hold the requirement):
```java
public Command kill() {
    return Commands.runOnce(() -> intake.openLoop(0), intake);
}
```

**`hold()`** — continuously command zero output (holds the subsystem requirement):
```java
public Command hold() {
    return Commands.run(() -> intake.openLoop(0), intake);
}
```

> Notice the difference between `kill` and `hold`: `runOnce` fires once and releases the requirement, `run` holds it indefinitely. Think about when you'd want each.

---

## Part 4: WristConstants

Create `WristConstants.java`. The Wrist is position-controlled, so it needs the full Motion Magic setup.

It needs:
- A `TalonFX` instance
- Gear ratio and conversion constants:
  - `kGearBoxRatio` — motor rotations per mechanism rotation
  - `kNativePosToDegrees` — `kGearBoxRatio * 360.0`
  - `kNativeVelToDPS` — same value (degrees per second per native velocity unit)
- `kMinDegree` and `kMaxDegree` — the mechanism's travel limits
- `kStowAngle` — a safe retracted position
- PID gains (`kP`, `kI`, `kD`, `kV`)
- Motion Magic parameters (cruise velocity, acceleration, expo gains)
- Current limits
- Neutral mode and invert direction

Wire it up in a `static {}` block the same way as `IntakeConstants`, but also apply `Slot0Configs` and `MotionMagicConfigs`:

```java
kMasterSlot0.kP = masterKP;
kMasterSlot0.kI = masterKI;
kMasterSlot0.kD = masterKD;
kMasterSlot0.kV = 0.03;

kMasterMotionMagic.MotionMagicCruiseVelocity = kMaxVelocityTicks;
kMasterMotionMagic.MotionMagicAcceleration = kMaxAccelerationTicks;
kMasterMotionMagic.MotionMagicExpo_kA = 0.1;
kMasterMotionMagic.MotionMagicExpo_kV = 0.1;
```

Also add soft limits so the motor controller enforces your angle bounds in hardware:

```java
kMasterSoftLimit.ForwardSoftLimitEnable = true;
kMasterSoftLimit.ForwardSoftLimitThreshold = kMaxDegree / kNativePosToDegrees;
kMasterSoftLimit.ReverseSoftLimitEnable = true;
kMasterSoftLimit.ReverseSoftLimitThreshold = kMinDegree / kNativePosToDegrees;
```

> Soft limits are enforced by the motor controller itself, not your code. If your code has a bug and sends an out-of-range setpoint, the motor still won't go past them. Always set them slightly inside your mechanical hard stops.

---

## Part 5: Wrist Subsystem

Create `Wrist.java` in `subsystems/`. It should `extend TalonFXSubsystem`.

The Wrist is more involved than the Intake because it has position control and a 2D visualization for simulation.

### Fields

Add angle bounds:

```java
private final double minAngle;
private final double maxAngle;
```

### Constructor

Call `super(master, velocityConversion, positionConversion, nominalVoltage, kDt)` and store your angle bounds.

### Methods

**`setAngle`** — clamp the angle and call `setPositionExpoVoltage`:
```java
public void setAngle(double angle) {
    double clampedAngle = MathUtil.clamp(angle, minAngle, maxAngle);
    setPositionExpoVoltage(clampedAngle, 0);
}
```

**`setEncoder`** — clamp before setting:
```java
public void setEncoder(double angle) {
    super.setEncoder(MathUtil.clamp(angle, minAngle, maxAngle));
}
```

**`angleReached`**:
```java
public boolean angleReached(double angle, double threshold) {
    return Math.abs(super.getPosition() - angle) < threshold;
}
```

Override `getName()` to return `"Wrist"`.

---

## Part 6: WristCommands

Create `WristCommands.java` in `commands/`. Take a `Wrist` in the constructor.

**`setAngle(double angle)`** — run until the wrist reaches the target:
```java
public Command setAngle(double angle) {
    return Commands.run(() -> wrist.setAngle(angle), wrist)
            .until(() -> wrist.angleReached(angle, 2));
}
```

**`setAngle(DoubleSupplier angleSupplier)`** — continuously track a dynamic setpoint:
```java
public Command setAngleIK(DoubleSupplier angleSupplier) {
    return Commands.run(() -> wrist.setAngle(angleSupplier.getAsDouble()), wrist)
            .until(() -> wrist.angleReached(angleSupplier.getAsDouble(), 2));
}
```

**`down()`** — go to `kMinDegree` (deployed).

**`up()`** — go to `kStowAngle` (retracted).

**`idle()`** — `Commands.idle(wrist)`.

---

## Part 7: RobotContainer

Instantiate both subsystems and their command classes:

```java
final Intake intake = new Intake(
    IntakeConstants.kMaster,
    1, 1,
    IntakeConstants.kNominalVoltage,
    GlobalConstants.kDt
);
final IntakeCommands intakeCommands = new IntakeCommands(intake);

final Wrist wrist = new Wrist(
    WristConstants.kMaster,
    WristConstants.kNativeVelToDPS,
    WristConstants.kNativePosToDegrees,
    WristConstants.nominalVoltage,
    GlobalConstants.kDt,
    WristConstants.kMinDegree,
    WristConstants.kMaxDegree
);
final WristCommands wristCommands = new WristCommands(wrist);
```

Bind buttons to test both subsystems:
- A button that runs `intakeCommands.intake()` while held, `intakeCommands.kill()` on release
- A button that runs `wristCommands.down()`, another that runs `wristCommands.up()`

---

## Checklist Before Moving On

- [ ] `IntakeConstants` has a motor instance, current limits, and motor output config applied in a `static {}` block
- [ ] `Intake` extends `TalonFXSubsystem` and has an `openLoop` method
- [ ] `IntakeCommands` has `intake()`, `kill()`, and `hold()` — and you understand the difference between `runOnce` and `run`
- [ ] `WristConstants` has gear ratio, conversion factors, PID gains, Motion Magic params, and soft limits
- [ ] `Wrist` extends `TalonFXSubsystem` and clamps angles in `setAngle`
- [ ] `WristCommands` has `setAngle(double)`, `down()`, `up()`, and `idle()`
- [ ] Intake spins when commanded and stops on kill
- [ ] Wrist reaches commanded angles and `angleReached` returns true at the target

**Tell a lead when you're done so they can verify both mechanisms work on the robot.**
