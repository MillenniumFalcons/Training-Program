# AdvantageKit Setup Guide

AdvantageKit is a logging framework by team 6328. It records everything your robot does to a log file that you can replay later in AdvantageScope to debug what happened during a match.

---

## Installation

**1. Add the vendor library**

Option 1: Find the vendor dependencies on the left side panel and click install advantagekit.

Option 2: Open the WPILib command palette (`Ctrl+Shift+P`) → **Manage Vendor Libraries → Install new library (online)** and paste:
```
https://github.com/Mechanical-Advantage/AdvantageKit/releases/latest/download/AdvantageKit.json
```

**2. Update `build.gradle`**

AdvantageKit uses an annotation processor to generate logging boilerplate. Add these two lines to the `dependencies` block in `build.gradle`:

```groovy
dependencies {
    // ... existing dependencies ...
    annotationProcessor "org.littletonrobotics.akit:akit-annotation-processor:$akit_version"
}
```

Then add the version variable near the top of the file where other versions are defined:

```groovy
def akit_version = "4.1.0" // check GitHub for the latest version
```

Run a Gradle build (`Ctrl+Shift+P` → **Build Robot Code**) to confirm everything compiles.

If you have AutoLogged classes, you may need to Clean Java Language Server Workspace (`Ctrl+Shift+P` → **Clean Java Language Server Workspace**) to get it to recognize the new classes.

---

## Setting Up the Logger

**1. Extend `LoggedRobot` instead of `TimedRobot`**

In `Robot.java`, change the class declaration:

```java
// Before
public class Robot extends TimedRobot {

// After
public class Robot extends LoggedRobot {
```

**2. Start the logger in `robotInit()`**

```java
@Override
public void robotInit() {
    if (isReal()) {
        Logger.addDataReceiver(new WPILOGWriter());   // save to USB drive on the robot
        Logger.addDataReceiver(new NT4Publisher());    // stream live to AdvantageScope
    } else {
        Logger.addDataReceiver(new NT4Publisher());
    }
    Logger.start();

    // ... rest of robotInit
}
```

> Always have a USB drive plugged into the RoboRIO during matches — that's how you get the log file back afterward.

---

## Logging Values

Call `Logger.recordOutput()` anywhere in your code to log a value:

```java
Logger.recordOutput("Drivetrain/Pose", periodicIO.pose);
Logger.recordOutput("Drivetrain/LeftDistanceMeters", getLeftDistanceMeters());
```

The key is a `/`-separated path — AdvantageScope organizes these into a folder tree. Call it inside `periodic()` or `writePeriodicOutputs()` so it runs every loop.

You can log primitives, WPILib structs (`Pose2d`, `Rotation2d`, etc.), and arrays of those types.

---

## Viewing in AdvantageScope

AdvantageScope comes bundled with the WPILib tools installation — no separate download needed.

- **Live:** Open  the AdvantageScope app → **File → Connect to Robot** → enter the robot IP (`10.36.47.2` for team 3647)
- **From a log file:** **File → Open Log** → open the `.wpilog` from the USB drive

Find your key in the left panel (e.g. `Drivetrain/Pose`) and drag it onto a **2D Field** or **3D Field** tab to visualize it.
