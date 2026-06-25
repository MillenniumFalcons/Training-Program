# Step 4: Odometry
*Timeline ~1 week.*

Odometry is a way of tracking your robot's position on the field by measuring how far each wheel has traveled. Every loop, you tell the odometry object "the left wheel moved this far, the right wheel moved this far, and the gyro says I'm facing this direction" — and it updates your estimated position accordingly.

It's like counting your steps to figure out where you are. It works well over short distances, but errors accumulate over time. In Step 5 you'll add vision to correct that drift.

> If you get stuck, ask a lead. There are no dumb questions.

---

## Setting Up the Gyro

The minibot uses an **ADIS16470 IMU** for its gyro. Follow the [ADIS gyro setup guide](Supplementals/Obscure%20Devices%20Setup%20Guide.md#adis-gyro-minibot-gyro) to initialize it, then add a helper method to `Drivetrain.java` so the rest of your code can easily get the heading:

```java
// In Drivetrain.java
private final ADIS16470_IMU gyro = new ADIS16470_IMU();

public Rotation2d getHeading() {
    return Rotation2d.fromDegrees(gyro.getAngle());
}
```

> `Rotation2d` is WPILib's way of representing an angle. It handles degree/radian conversion for you. `fromDegrees()` creates one from a degree value.

---

## Unit Conversion: Motor Rotations → Meters

The SparkMax encoder measures position in **motor rotations**, but odometry needs **meters traveled**. There are two things to account for:

- The gearbox has a **5:1 ratio** — the motor spins 5 times for every 1 wheel rotation
- The wheel radius is **2 inches (0.0508 meters)**, so one full wheel rotation = `2π × 0.0508` meters of travel

The formula is:

> **distance (m) = (motor rotations ÷ gear ratio) × 2π × wheel radius (m)**

Add these constants to `DrivetrainConstants.java`:

```java
public static final double GEAR_RATIO = 5.0;
public static final double WHEEL_RADIUS_METERS = 0.0508; // 2 inches
```

Then add helper methods in `Drivetrain.java` to get the distance each side has traveled:

```java
private double getLeftDistanceMeters() {
    return (leftMotor.getEncoder().getPosition() / DrivetrainConstants.GEAR_RATIO)
           * 2 * Math.PI * DrivetrainConstants.WHEEL_RADIUS_METERS;
}

private double getRightDistanceMeters() {
    return (rightMotor.getEncoder().getPosition() / DrivetrainConstants.GEAR_RATIO)
           * 2 * Math.PI * DrivetrainConstants.WHEEL_RADIUS_METERS;
}
```

---

## Setting Up Odometry

WPILib's `DifferentialDriveOdometry` class tracks your position given left/right wheel distances and a gyro heading.

### In `Drivetrain.java`

Add the odometry object as a field. The constructor takes:
1. Your starting gyro heading (should be `0` when the robot boots)
2. Starting left distance (always `0`)
3. Starting right distance (always `0`)
4. Starting pose — where on the field the robot begins (use `new Pose2d()` for the origin)

```java
private final DifferentialDriveOdometry odometry = new DifferentialDriveOdometry(
    getHeading(),
    0, 0,
    new Pose2d()
);
```

Also add a field in `periodicIO` to store the current pose:

```java
public Pose2d pose = new Pose2d();
```

### Calling `update()` Every Loop

In your `readPeriodicInputs()` method, call `odometry.update()` every robot loop (50Hz). It takes the current gyro heading and the total distance each side has traveled since the robot turned on:

```java
@Override
public void readPeriodicInputs() {
    periodicIO.pose = odometry.update(
        getHeading(),
        getLeftDistanceMeters(),
        getRightDistanceMeters()
    );
}
```

> **Why total distance and not change-in-distance?** The odometry object keeps track of the previous position internally — you just give it the raw encoder reading each loop and it handles the math. This is why the distance methods read directly from the encoder position rather than calculating a delta.

Add a getter so other parts of the code can read the pose:

```java
public Pose2d getPose() {
    return periodicIO.pose;
}
```

---

## Logging the Pose

To visualize your pose, use AdvantageKit to log it. If you haven't set it up yet, follow the [AdvantageKit Setup Guide](Supplementals/AdvantageKit%20Setup%20Guide.md) first.

Once the logger is running, call `Logger.recordOutput()` in `periodic()` or `writePeriodicOutputs()`:

```java
Logger.recordOutput("Drivetrain/Pose", periodicIO.pose);
```

Open **AdvantageScope**, connect to the robot (or open a log file), find `Drivetrain/Pose` in the left panel, and drag it onto a 2D field view. You should see a dot representing your robot move as you drive.

---

## Testing

Drive around and verify the pose looks correct:
- Drive straight forward ~1 meter and check the pose moved roughly 1 meter
- Spin in place and check the heading updates correctly
- Use a tape measure to sanity-check the distance calculation

**Fix any conversion errors before moving on to Step 5.**

---

## Limitations of Odometry

Odometry alone is unreliable for a full match. It breaks down when:
- The robot accelerates hard and the wheels slip
- Another robot hits you
- Anything else causes wheels to spin without the robot actually moving

Since odometry just counts wheel rotations, any wheel slip causes the estimated position to drift from reality — and the error accumulates over time. By the end of a 15-second auto, the robot can be off by half a meter or more.

In Step 5, you'll add AprilTag vision to correct this drift.

---

## Checklist Before Moving On

- [ ] Gyro is initialized and `getHeading()` returns a correct `Rotation2d`
- [ ] Distance conversion constants (`GEAR_RATIO`, `WHEEL_RADIUS_METERS`) are in `DrivetrainConstants`
- [ ] `getLeftDistanceMeters()` and `getRightDistanceMeters()` return sensible values
- [ ] `DifferentialDriveOdometry` is initialized with heading, distances, and starting pose
- [ ] `update()` is called every loop in `readPeriodicInputs()`
- [ ] AdvantageKit is set up and `Logger.start()` is called in `robotInit()`
- [ ] Pose is logged with `Logger.recordOutput` and visible in AdvantageScope
- [ ] Driving ~1 meter moves the pose ~1 meter on the field view

---

**Tell a lead when you're done so they can verify your pose looks correct before you move on.**
