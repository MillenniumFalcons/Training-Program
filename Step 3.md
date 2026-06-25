# Step 3: Turn the Wheel 90 Degrees
*Timeline ~1 week.*

Now it's time to learn about PID control. In this step, you'll program one of the minibot's wheels to turn to a target position. You'll implement it **twice** — first using WPILib's `PIDController` class to understand how PID works, then using the SparkMax's onboard PID controller, which is how it's done in practice.

> If you get stuck, reference the finished mini-bot project [here](https://github.com/ChrisC12345/mini-bot) — but try to work through it yourself first.

---

## Background: What is PID?

A PID controller is an algorithm that drives a system to a target state (called the **setpoint**) by continuously measuring the current state and adjusting the output to close the gap.

The formula is: **output = kP × e(t) + kI × ∫e(t) + kD × e'(t)**

Where **e(t)** is the error — the difference between the setpoint and the current measurement. Here's what each term does:

- **kP (Proportional):** The main driver. Output is proportional to the error — the further you are from the setpoint, the harder the motor pushes. As you get close, output shrinks.
- **kD (Derivative):** Dampens oscillations. It looks at how fast the error is changing — if you're approaching the setpoint quickly, it starts reducing the output early to prevent overshooting.
- **kI (Integral):** Corrects for steady-state error that P alone can't fix. In practice, leave this at 0 unless you have a specific reason — it often causes more problems than it solves.

### A worked example

Say you're trying to move to a position of 5 meters, starting from 0, with kP=1, kI=0, kD=0:

- At the start, error = 5, so output = 1 × 5 = **5 volts**
- After moving to 3 meters, error = 2, so output = **2 volts**
- As you get closer, output keeps decreasing — the motor slows naturally as it approaches the target

The issue with setting kI and kD to 0 is that the motor will overshoot because as it approaches the target, the output will still give you positive voltage which will keep accelerating the motor up to a certain point. 
- Even at the setpoint, the PID controller outputs 0 volts, but voltage is not proportional to velocity so it can't instantly stop the motor. This is called **overshoot**, and it causes the motor to oscillate. 
- To fix this, we need to add kD to dampen the oscillations.

### Tuning process

Before trying it on the robot, visit [this interactive tuning guide](https://www.luisllamas.es/en/pid-controller-simulator/) to get some practice. Set the mode to manual under Simulation Control and under PID improvements uncheck everything. 
- You're going to need kI because the simulation simulates a resistive force (e.g. gravity, friction) which will stop the motor from reaching the setpoint. This is called **steady-state error**.

1. Set kI and kD to 0. Increase kP until the wheel reaches the setpoint reliably.
2. If it oscillates, lower kP until it stops, then add a small kD to dampen any remaining overshoot.
3. Leave kI at 0 unless the wheel consistently stops slightly short of the target.

---

## Unit Conversion: Motor Rotations → Wheel Degrees

The SparkMax encoder measures position in **motor rotations**, but we want to control the wheel in **degrees**. There are two things to account for:

- The minibot's gearbox has a **5:1 ratio** — for every 1 wheel rotation, the motor rotates 5 times.
- A full wheel rotation is **360 degrees**.

So to convert a target angle in degrees to motor rotations:

> **motor rotations = (degrees / 360) × gear ratio**

You'll need to use this conversion whenever you set a position target, and also when reading the current position back from the encoder.

---
**There are two methods for implementing PID control in WPILib. The following two sections will go over both methods, I want you to do each one separately.** As a bonus, try implementing the PID from scratch, doing all the calculations in the code.

## Method 1: WPILib PIDController

WPILib's `PIDController` class runs the PID loop in your robot code. You give it the current measurement and the setpoint each loop, and it returns the output to send to the motor.

### In `Drivetrain.java`

Add a `PIDController` field, initialized with starting gains of `(0, 0, 0)` — you'll tune these on the robot:

```java
private final PIDController pid = new PIDController(0, 0, 0);
```

Add a `setAngle(double degrees)` method that:
- Converts the target angle from degrees to motor rotations
- Reads the current motor position from the encoder
- Calls `pid.calculate(currentPosition, targetRotations)` to get the output
- Stores that output in `periodicIO.leftOutput` with `ControlType.kDutyCycle`

> **Note:** `pid.calculate()` returns a duty cycle value (-1.0 to 1.0) that you send directly to the motor. You're still using `kDutyCycle` here because the PID loop is running in your code and outputting a power percentage — it's the PIDController doing the position math, not the SparkMax.

### In `DrivetrainCommands.java`

Add a `turnWheel()` command that turns the wheel 90 degrees. Use the full command class syntax (not `Commands.run`) so you can use `initialize()`, `execute()`, and `isFinished()` separately:

- In `initialize()`: store the target angle (current wheel position + 90 degrees, converted to motor rotations) as a local variable.
- In `execute()`: call `drivetrain.setAngle(target)` each loop.
- In `isFinished()`: return `true` when the motor position is within a small tolerance of the target (e.g., ±0.05 rotations).

> **Why store the target in `initialize()` and not `execute()`?** `initialize()` runs once when the command starts. If you calculated the target inside `execute()`, it would use the motor's current (moving) position as the new base every loop — the target would keep shifting and the wheel would never stop.

### In `RobotContainer.java`

Bind the command to a button. For example, to run it when A is pressed:

```java
mainController.buttonA.onTrue(drivetrainCommands.turnWheel());
```

**Test this on the robot and tune your kP until the wheel reliably turns 90 degrees before moving on.**

---

## Method 2: SparkMax Onboard PID

The SparkMax has a PID controller built into the motor controller itself, running at **1000Hz** — much faster than the robot loop's 50Hz. This gives smoother, more responsive control and is what we actually use in the mini-bot.

Instead of calculating the output in your robot code and sending a duty cycle, you give the SparkMax a target position in motor rotations and let it handle the math internally.

### Configure the gains in `DrivetrainConstants.java`

Set the PID gains on the `SparkMaxConfig` before passing it to the motor. These are stored on the controller itself:

```java
configLeft.closedLoop.p(0.2);
configLeft.closedLoop.i(0);
configLeft.closedLoop.d(0);
```

You can remove the `PIDController` field from `Drivetrain` — the SparkMax handles it now.

### Update `Drivetrain.java`

Add a `turn(double setPoint)` method that:
- Sets `periodicIO.controlType` to `ControlType.kPosition`
- Sets `periodicIO.leftOutput` to the target in motor rotations

When `writePeriodicOutputs()` calls `setReference()` with `ControlType.kPosition`, the SparkMax interprets the value as a target position and runs its internal PID to get there — you don't calculate the output yourself.

> **Comparison:** With Method 1, your code calculates the motor output every 20ms. With Method 2, the SparkMax runs the loop every 1ms using the target you gave it. The result is the same behavior, but Method 2 is faster and more precise.

### Update `DrivetrainCommands.java`

Update your `turnWheel()` command to call `drivetrain.turn(setPoint)` in `execute()` instead of `drivetrain.setAngle()`. The `initialize()` and `isFinished()` logic stays the same.

---

## Checklist Before Testing

- [ ] Unit conversion from degrees to motor rotations is correct (degrees / 360 × 5)
- [ ] **Method 1:** `PIDController` is initialized in `Drivetrain`, `setAngle()` calls `pid.calculate()` and sets `kDutyCycle`
- [ ] **Method 2:** PID gains are set in `DrivetrainConstants`, `turn()` sets `controlType` to `kPosition`
- [ ] `turnWheel()` stores the setpoint in `initialize()`, not `execute()`
- [ ] `isFinished()` returns `true` when within tolerance of the setpoint
- [ ] Command is bound to a button in `RobotContainer`
- [ ] Code builds with no errors

---

**Tell a lead when you're done so they can help you tune the PID gains on the real robot.**
