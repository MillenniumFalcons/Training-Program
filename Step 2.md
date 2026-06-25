# Step 2: Minibot Drive
*Timeline ~2 weeks.*

> **Before you start:** Make sure you've completed Step 1. Follow this guide: [Intro to WPILib](Supplementals/Intro-to-Wpilib.md) to install WPILib VSCode and create your mini-bot project. If you get stuck at any point, you can reference the finished mini-bot project [here](https://github.com/ChrisC12345/mini-bot) — but try to figure it out yourself first. 

## Overview

By the end of this step, you'll be able to drive the minibot with an Xbox controller. You'll build three things:

1. **`DrivetrainConstants.java`** — stores the motor objects and their configuration
2. **`Drivetrain.java`** — the subsystem that controls the motors
3. **`DrivetrainCommands.java`** — the command that reads joystick input and calls the drive method
4. **Wire it all together in `RobotContainer.java`**

Before you start coding open the [WPILib guide to command based programming](https://docs.wpilib.org/en/stable/docs/software/commandbased/index.html) and skim through the sections titled "What is Command Based Programming", "Commands", "Subsystems", and "Structuring a Command-Based Robot Project". You do not need to understand everything, just get an idea of what's going on. Reference this guide throughout Step 2 when you are confused. 

If you ever get confused, please ask me or an assistant head for help. If we're not available Claude, Gemini, and ChatGPT are all excellent teachers too.

**IMPORTANT:** Don't copy paste anything into your code unless the instructions explicitly say so; you need to type everything out yourself. This is for you to develop experience with programming.


## Part 1: DrivetrainConstants

### 1.1 — Create `DrivetrainConstants.java`

In your `constants` folder, create a new file called `DrivetrainConstants.java`. This file holds the motor objects and their configuration so that `Drivetrain.java` stays clean and all the hardware setup is in one place.

> **Don't have RevLib installed?** You need to install it before `SparkMax` will be recognized. See [How to Run NEO Motors](Supplementals/How%20to%20run%20NEO%20motors.md) for instructions.

Inside the class, create two `public static final SparkMax` fields — one for the left motor and one for the right. A `SparkMax` takes two arguments:
- The **CAN ID** — a number that identifies which motor controller you're talking to. Ask a lead what IDs your robot uses, or find them using [Rev Hardware Client](https://docs.revrobotics.com/rev-hardware-client).
- `MotorType.kBrushless` — use this for all NEO motors.

```java
public static final SparkMax leftMotor = new SparkMax(5, MotorType.kBrushless);
```

### 1.2 — Configure the Motors

Create a `SparkMaxConfig` object for each motor. Then use a **static initializer block** (`static { ... }`) to apply the configuration. A static initializer runs once when the class is first loaded — it's the right place to configure motors since they only need to be set up once.

There are two important things to configure:

- **Inversion:** The left and right motors face opposite directions on the robot. Without fixing this, sending a positive value to both sides would make them fight each other instead of driving forward. Use `.inverted(true)` on one side's config to flip it so both sides drive "forward" together. You may need to experiment with which side to invert.

- **Idle Mode:** Set both motors to `IdleMode.kBrake`. This means when no command is running, the motors will resist being moved — stopping the robot from rolling freely.

Once your configs are set up, apply them to each motor:
```java
leftMotor.configure(configLeft, ResetMode.kResetSafeParameters, PersistMode.kPersistParameters);
```

---

## Part 2: The Drivetrain Subsystem

### 2.1 — Create `Drivetrain.java`

In your `subsystems` folder, create a new file called `Drivetrain.java`.

Copy the `PeriodicSubsystem` interface into your `lib` folder from the [mini-bot repo](https://github.com/ChrisC12345/mini-bot/blob/master/src/main/java/team3647/lib/PeriodicSubsystem.java). Your class should `implement PeriodicSubsystem`.

If VS Code underlines the interface name in red, hover over it and select **"Add unimplemented methods"** — it will generate the method stubs for you.

### 2.2 — Add `PeriodicIO`

Inside your `Drivetrain` class, create a `public static` inner class called `PeriodicIO`. This is a simple container that holds the values you want to send to each motor each loop. IO stands for Input/Output. Give it three fields:
- `double leftOutput` — speed for the left side (-1.0 to 1.0)
- `double rightOutput` — speed for the right side (-1.0 to 1.0)
- `ControlType controlType` — tells the SparkMax how to interpret the output values. Default this to `ControlType.kDutyCycle`, which means percentage of full power.

Then create an instance of `PeriodicIO` as a private field in `Drivetrain`.

> **Why `PeriodicIO`?** It keeps all the "state" of the subsystem in one place. Instead of scattering variables around the class, everything the motors need is in `periodicIO`. Methods like `drive()` update `periodicIO`, and `writePeriodicOutputs()` sends it to the hardware.

### 2.3 — Add the Constructor

Add two `private final SparkMax` fields. Then write a constructor that takes a `SparkMax` for each side as parameters and assigns them. Inside the constructor, reset both encoders to 0 using `getEncoder().setPosition(0)` so they start fresh every boot.

The motors themselves are created in `DrivetrainConstants` and passed in here — this keeps all hardware configuration in the constants file.

### 2.4 — Override `writePeriodicOutputs()`

This method is called automatically every robot loop (~50 times per second). Override it to send the current `periodicIO` values to the motors.

To call `setReference()`, first get the closed loop controller from the motor, then call it with the output value and control type stored in `periodicIO`:

```java
leftMotor.getClosedLoopController().setReference(periodicIO.leftOutput, periodicIO.controlType);
```

Do this for both motors.

### 2.5 — Write the `drive()` Method

Create a `public void drive(double forward, double rotation)` method. This is what commands will call to move the robot.

Inside, use `DifferentialDrive.arcadeDriveIK(forward, rotation, false)` to convert the forward/rotation inputs into individual left and right wheel speeds. This returns a `WheelSpeeds` object with `.left` and `.right` fields — store those in `periodicIO`, and set `periodicIO.controlType` to `ControlType.kDutyCycle`.

> **Tip:** If the robot spins when you push forward (or vice versa), try negating the `rotation` argument passed into `arcadeDriveIK`. The joystick axis may be reversed compared to what the method expects.

### 2.6 — Add `getName()`

`PeriodicSubsystem` requires a `getName()` method. Just have it return the string `"drivetrain"`.

---

## Part 3: The Drive Command

### 3.1 — Create `DrivetrainCommands.java`

In your `commands` folder, create `DrivetrainCommands.java`.

> **Not sure what a command is?** Read [How to Make a Command](Supplementals/How-to-Make-a-Command.md) and [WPILib's command-based guide](https://docs.wpilib.org/en/stable/docs/software/commandbased/index.html) before continuing.

Give the class a `private final Drivetrain` field, and a constructor that takes a `Drivetrain` and stores it.

### 3.2 — Write the `drive` Command

Write a method called `drive` that returns a `Command` and takes two `DoubleSupplier` parameters — one for speed, one for rotation.

> **Why `DoubleSupplier` instead of `double`?** A plain `double` is frozen at the value it had when the command was created. A `DoubleSupplier` produces a fresh value every time you call `.getAsDouble()` — so the command always reads the *current* joystick position each loop.

Use `Commands.run(...)` to create the command. Inside the lambda, call `drivetrain.drive(...)` passing in `.getAsDouble()` on each supplier:

```java
return Commands.run(
    () -> drivetrain.drive(speed.getAsDouble(), rotation.getAsDouble()),
    drivetrain
);
```

The second argument `drivetrain` declares the drivetrain as a **requirement**, which prevents two commands from trying to drive at the same time.

---

## Part 4: Wiring It Together in RobotContainer

Open `RobotContainer.java`.

### 4.1 — Add the Joysticks class

The `Joysticks` class wraps an Xbox controller and gives you easy access to stick values and button presses. It's not part of WPILib — copy it from the mini-bot repo into your `lib/inputs` folder:

[Copy Joysticks.java from here](https://github.com/ChrisC12345/mini-bot/blob/master/src/main/java/team3647/lib/inputs/Joysticks.java)

### 4.2 — Declare your objects

As fields at the top of `RobotContainer` (outside any method), create:
- A `Drivetrain` object, passing in `DrivetrainConstants.leftMotor` and `DrivetrainConstants.rightMotor`
- A `DrivetrainCommands` object, passing in your drivetrain
- A `Joysticks` object with port `0` (port 0 = first controller plugged into the driver station)

### 4.3 — Register and set the default command
Read through this: https://docs.wpilib.org/en/stable/docs/software/commandbased/binding-commands-to-triggers.html before continuing.

Inside the `RobotContainer` constructor:

1. Call `CommandScheduler.getInstance().registerSubsystem(drivetrain)`. This tells the scheduler to call `periodic()` on the drivetrain every loop, which is what makes `writePeriodicOutputs()` run automatically.

2. Call `drivetrain.setDefaultCommand(...)` and pass in `drivetrainCommands.drive(...)`. A **default command** runs whenever nothing else is using the subsystem — since driving is the default behavior, this is where it belongs.

3. For the `DoubleSupplier` arguments, pass `mainController::getLeftStickY` for speed and `mainController::getRightStickX` for rotation. The `::` syntax is a **method reference** — shorthand for `() -> mainController.getLeftStickY()`. It creates a `DoubleSupplier` that reads the joystick live each loop. 
Learn about java lambdas here: https://www.w3schools.com/java/java_lambda.asp.

---

## Checklist Before Testing

- [ ] RevLib is installed (no red underlines on `SparkMax`)
- [ ] `DrivetrainConstants.java` creates both motors and applies configuration
- [ ] `Drivetrain.java` implements `PeriodicSubsystem`
- [ ] `writePeriodicOutputs()` calls `setReference` on both motors
- [ ] `drive()` uses `arcadeDriveIK` and updates `periodicIO`
- [ ] `DrivetrainCommands.java` has a `drive` command using `DoubleSupplier`
- [ ] `Joysticks.java` is copied into your `lib/inputs` folder
- [ ] `RobotContainer` registers the drivetrain and sets the default drive command
- [ ] Code builds with no errors (`Ctrl+Shift+P` → `WPILib: Build Robot Code`)

To test the code you will need to install FRC Driver Station. Follow this guide: https://docs.wpilib.org/en/stable/docs/zero-to-robot/step-2/frc-game-tools.html.

Read this to learn how to use FRC Driver Station: https://docs.wpilib.org/en/stable/docs/software/driverstation/driver-station.html.

**When you've done this, ask a lead to help test your code on the minibot.**
