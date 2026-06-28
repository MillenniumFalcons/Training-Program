# Step 6: Swerve
*Timeline ~2 weeks.*

**What is swerve?**
Swerve is a drivetrain where each wheel can rotate independently, allowing the robot to spin and translate simultaneously. It's what we use on most of our competition robots. 

Ask a lead if you're not sure what the swerve is.

> **NOTE:**
> Create a new project for this exercise and install the CTRE Phoenix 6 vendor library.

Skim through the file "How to Structure a Custom Swerve Drivetrain.md" to get the basic idea of how swerve works.

There is a sample project in the [Supplementals](Supplementals/Example%20Projects/Sample-Swerve-Code.md) folder. You really should use this only as a last resort, try to figure everything out yourself or ask a lead for help. Do not copy it.

---

## Part 1: Configure the Drivetrain

We don't manually configure swerve modules — CTRE's Tuner X does it for us.

1. Run the **Swerve Project Generator** in Tuner X ([docs](https://v6.docs.ctr-electronics.com/en/stable/docs/tuner/tuner-swerve/index.html))
2. When done, click **"Only Generate TunerConstants"** and save the file to your `constants/` folder
3. Also click **"Save as JSON"** and save that file somewhere safe as a backup

The generated `TunerConstants.java` contains all your hardware IDs, gear ratios, PID gains, and a nested class called `TunerSwerveDrivetrain` — a typed wrapper around CTRE's `SwerveDrivetrain`.

---

## Part 2: The SwerveDrive Subsystem

Create `SwerveDrive.java` in `subsystems/`. This class should **extend `TunerConstants.TunerSwerveDrivetrain`** and implement `PeriodicSubsystem`.

> **IMPORTANT:**
> Read the [CTRE Swerve API overview](https://v6.docs.ctr-electronics.com/en/stable/docs/api-reference/mechanisms/swerve/swerve-overview.html) before writing this class. Focus on how `SwerveDrivetrain`, `SwerveRequest`, and telemetry work.

### PeriodicIO

Add a `PeriodicIO` inner class to hold all your state. At minimum you need:
- A **master `SwerveRequest`** — this is what gets sent to the drivetrain each loop, initialized as `SwerveRequest.Idle`
- A **`FieldCentric`** request — for driver-relative driving
- A **`RobotCentric`** request — for robot-relative driving

```java
public class PeriodicIO {
    public SwerveRequest masterRequest = new SwerveRequest.Idle();
    public SwerveRequest.FieldCentric fieldCentric = new SwerveRequest.FieldCentric();
    public SwerveRequest.RobotCentric robotCentric = new SwerveRequest.RobotCentric();
}
```

### Constructor

The constructor takes a `SwerveDrivetrainConstants` and varargs `SwerveModuleConstants<?, ?, ?>...` and passes them to `super()`. After calling `super()`:
- Call `registerTelemetry(this::setStates)` to wire up state logging
- Reset the pose and gyro to zero

```java
public SwerveDrive(
        SwerveDrivetrainConstants drivetrainConstants,
        SwerveModuleConstants<?, ?, ?>... modules) {
    super(drivetrainConstants, modules);
    registerTelemetry(this::setStates);
}
```

### Telemetry Callback

`registerTelemetry` requires a method that accepts a `SwerveDriveState`. Use it to cache the current module states and pose for logging:

```java
public void setStates(SwerveDriveState state) {
    periodicIO.states = state.ModuleStates;
    periodicIO.target = state.ModuleTargets;
    periodicIO.pose = state.Pose;
}
```

### readPeriodicInputs / writePeriodicOutputs

- `readPeriodicInputs`: log your pose, module states, and module targets with AdvantageKit
- `writePeriodicOutputs`: send `periodicIO.masterRequest` to the hardware by calling `setControl(periodicIO.masterRequest)`

### Drive Methods

Each drive method modifies the appropriate specific request, then sets `masterRequest` equal to it. `writePeriodicOutputs` sends it to the hardware.

**Field-relative drive** (most common for teleop):
```java
public void driveFieldCentric(double vx, double vy, double rotationRate) {
    periodicIO.fieldCentric
        .withVelocityX(vx)
        .withVelocityY(vy)
        .withRotationalRate(rotationRate);
    periodicIO.masterRequest = periodicIO.fieldCentric;
}
```

**Robot-relative drive**:
```java
public void driveRobotCentric(double vx, double vy, double rotationRate) {
    periodicIO.robotCentric
        .withVelocityX(vx)
        .withVelocityY(vy)
        .withRotationalRate(rotationRate);
    periodicIO.masterRequest = periodicIO.robotCentric;
}
```

---

## Part 3: DrivetrainCommands

Create `DrivetrainCommands.java` in `commands/`. This is pretty much the same as the minibot's `DrivetrainCommands`.

You should be able to figure this out yourself.

Scale your velocity inputs by the robot's max speed (e.g. `TunerConstants.kSpeedAt12Volts`). For field-relative drive, flip the sign when on the Red alliance so the driver's "forward" always matches their perspective.

---

## Part 4: RobotContainer

Instantiate `SwerveDrive` with the constants from `TunerConstants`:

```java
final SwerveDrive swervedrive = new SwerveDrive(
    TunerConstants.DrivetrainConstants,
    TunerConstants.FrontLeft,
    TunerConstants.FrontRight,
    TunerConstants.BackLeft,
    TunerConstants.BackRight
);
```

Register it's default command.

---

## Checklist Before Moving On

- [ ] `TunerConstants.java` is generated and in your `constants/` folder
- [ ] `SwerveDrive` extends `TunerConstants.TunerSwerveDrivetrain` and implements `PeriodicSubsystem`
- [ ] `PeriodicIO` has a master request, a `FieldCentric` request, and a `RobotCentric` request
- [ ] `registerTelemetry` is called in the constructor
- [ ] Pose and module states are logged in AdvantageScope
- [ ] `writePeriodicOutputs` calls `setControl(periodicIO.masterRequest)`
- [ ] Field-relative and robot-relative drive methods work correctly
- [ ] `DrivetrainCommands` wraps both drive methods with `DoubleSupplier` parameters
- [ ] `RobotContainer` instantiates the drivetrain and sets the default command
- [ ] Robot drives correctly in field-relative mode from a joystick

**Tell a lead when you're done so they can verify the robot drives correctly.**
