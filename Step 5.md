# Step 5: Vision
*Timeline ~2 weeks.*

> This step is a little more advanced. If you want to start programming the actual robot, skip to Step 6; it is more relevant to the work you'll actually be doing during the season.

In Step 4 you set up odometry, which tracks position by counting wheel rotations. The problem is that any wheel slip causes the estimate to drift, and over a full match that error adds up fast.

In this step, you'll add **AprilTag vision** using the Limelight to get direct position measurements from the field, then fuse them with your odometry using a Kalman Filter to get a pose estimate that's both continuous and accurate.

> If you get stuck, ask a lead. There are no dumb questions.

---

## Configuring the Limelight

Before writing any code, you need to set up the Limelight through its web interface.

### Accessing the Web Interface

1. Power on the robot and connect your laptop to the robot's network (either via the radio or a direct ethernet cable)
2. Open a browser and go to `http://limelight.local:5801`
   - If that doesn't resolve, try `http://10.36.47.11:5801` (the Limelight's default IP on team 3647's network)
3. You should see a live camera feed and a settings panel

### Setting Up an AprilTag Pipeline

1. In the left panel, click **Create Pipeline** and select **AprilTag**
2. Set the **Tag Family** to `36h11` — that's the family used in FRC
3. Under **Camera**, tune the **exposure** down until the image looks crisp and the background isn't blown out. A darker image with clear tag edges works better than a bright one
4. Under **AprilTag**, enable **Do Multi-Target Estimation (MegaTag 1)** — this significantly reduces ambiguity when two tags are visible

### Setting the Camera Pose

The Limelight needs to know where it's mounted on the robot to correctly calculate field position. Under the **3D** tab:

- Set the camera's **X, Y, Z** offset from the robot center (in meters or inches, depending on the unit setting)
- Set the **roll, pitch, yaw** of the camera

> This can be done in code instead with `LimelightHelpers.setCameraPose_RobotSpace()`, which is what the mini-bot does. Either way is fine — just don't set it in both places or they'll conflict.

### Verify It's Working

Point the camera at an AprilTag on the field. You should see:
- A green box drawn around the tag in the camera feed
- A **3D pose** appearing in the "Poses" tab

If you don't see a detection, check that the pipeline type is set to AprilTag (not retroreflective or neural), and that the tag family matches.

---

## Limelight Quick Start

The minibot has a **Limelight** — a camera with an onboard computer that handles AprilTag detection and pose math for us.

### Getting the Library

The Limelight library is a single file called `LimelightHelpers.java`. Copy it from `team3647/lib/vision` in the most recent robot code repo and drop it into your project's `frc/robot` folder.

> You don't need to install anything — it's just a Java file you include directly.

### Reading a Pose

Call `LimelightHelpers.getBotPose2d_wpiBlue("")` to get the robot's estimated pose in field coordinates (origin = blue alliance corner, which is what WPILib uses internally):

```java
Pose2d visionPose = LimelightHelpers.getBotPose2d_wpiBlue("");
```

Log it with AdvantageKit and point the camera at an AprilTag — you should see a pose appear in AdvantageScope.

```java
Logger.recordOutput("Drivetrian/Vision Pose", visionPose);
```

Verify this is working before continuing.

---

## Integrating Vision with Odometry

We can't always see AprilTags during a match — robots and field elements will block the view. So we can't rely on vision alone. But we also can't rely on odometry alone because it drifts.

The solution is to **fuse them together**: use odometry as the baseline, and whenever we get a vision pose, use it to nudge the odometry estimate toward the real position.

We do this with a **Kalman Filter**, which WPILib implements in the `DifferentialDrivePoseEstimator` class. It works just like `DifferentialDriveOdometry` from Step 4, but also accepts vision measurements. It weights each source based on how much you tell it to trust it.

### Upgrading to `DifferentialDrivePoseEstimator`

First, add a `DifferentialDriveKinematics` constant to `DrivetrainConstants.java`. This describes your drivetrain's geometry — specifically the **trackwidth**, which is the distance between the center of the left wheels and the center of the right wheels. Measure this on the robot:

```java
// In DrivetrainConstants.java
public static final double TRACK_WIDTH_METERS = some number; // measure on your robot
public static final DifferentialDriveKinematics KINEMATICS =
    new DifferentialDriveKinematics(TRACK_WIDTH_METERS);
```

Now replace the `DifferentialDriveOdometry` field in `Drivetrain.java` with a `DifferentialDrivePoseEstimator`. The constructor takes the same arguments as before, with `kinematics` added as the first argument:

```java
private final DifferentialDrivePoseEstimator poseEstimator = new DifferentialDrivePoseEstimator(
    DrivetrainConstants.KINEMATICS,
    getHeading(),
    0, 0,
    new Pose2d()
);
```

Update `readPeriodicInputs()` to call `update()` on the estimator — the signature is identical to the old odometry call:

```java
periodicIO.pose = poseEstimator.update(
    getHeading(),
    getLeftDistanceMeters(),
    getRightDistanceMeters()
);
```

### Adding Vision Measurements

When you get a vision pose from the Limelight, call `addVisionMeasurement()` on the estimator. This requires three things:

1. **The pose** — the `Pose2d` from `getBotPose2d_wpiBlue`
2. **The timestamp** — *when* the image was taken, not when you received it. The Limelight takes time to process the image, so you need to subtract that latency out
3. **Standard deviations** — how much to trust the x, y, and rotation values from vision

#### Timestamp

```java
// Total latency = pipeline processing time + image capture time (both in ms, convert to seconds)
double latencySeconds = (LimelightHelpers.getLatency_Pipeline("")
                       + LimelightHelpers.getLatency_Capture("")) / 1000.0;
double timestamp = Timer.getFPGATimestamp() - latencySeconds;
```

> `Timer.getFPGATimestamp()` returns the current time in seconds. Subtracting the latency gives the time the image was actually taken. Getting this right matters — giving the estimator a measurement with the wrong timestamp causes it to apply the correction at the wrong point in the robot's path.

#### Standard Deviations

Standard deviations tell the Kalman filter how much to trust each part of the vision measurement. Lower = more trust, higher = less trust.

- **x and y:** Vision is reasonably accurate here. A value of `0.05` works well as a starting point.
- **rotation:** We have a gyro, which is far more accurate for heading than vision. Set this very high (e.g. `9999`) to effectively ignore the vision rotation entirely.

```java
// Trust x/y, ignore vision rotation since we have a gyro
VecBuilder.fill(0.05, 0.05, 9999)
```

#### Putting It Together

Add a method to `Drivetrain.java` that takes a vision pose and feeds it to the estimator:

```java
public void addVisionMeasurement(Pose2d visionPose, double timestamp) {
    poseEstimator.addVisionMeasurement(
        visionPose,
        timestamp,
        VecBuilder.fill(0.05, 0.05, 9999)
    );
}
```

Then, in `readPeriodicInputs()`, call it alongside your `update()` call:

```java
Pose2d visionPose = LimelightHelpers.getBotPose2d_wpiBlue("");
double latencySeconds = (LimelightHelpers.getLatency_Pipeline("")
                       + LimelightHelpers.getLatency_Capture("")) / 1000.0;
double timestamp = Timer.getFPGATimestamp() - latencySeconds;

addVisionMeasurement(visionPose, timestamp);
```

> In AdvantageScope, you should now see your odometry pose slowly drift toward the vision pose when you're looking at a tag. **If it doesn't, ask for help and fix it before continuing.**

---

## Filtering Bad Poses

There's a problem: when the Limelight can't see any tags, `getBotPose2d_wpiBlue` returns `(0, 0, 0)`. If we blindly feed that into the estimator, our pose will get pulled toward the origin of the field — which is obviously wrong.

We need to **filter out bad poses** before passing them to `addVisionMeasurement`.

<img src="Supplementals/images/filtering.png" alt="drawing" width="400" />

Wrap the vision measurement call in checks:

```java
Pose2d visionPose = LimelightHelpers.getBotPose2d_wpiBlue("");

boolean isZeroPose = visionPose.getX() == 0 && visionPose.getY() == 0;
boolean isOutsideField = visionPose.getX() < 0 || visionPose.getX() > 16.54
                      || visionPose.getY() < 0 || visionPose.getY() > 8.21;
boolean isInTheAir = LimelightHelpers.getBotPose3d_wpiBlue("").getZ() > 0.5;

if (!isZeroPose && !isOutsideField && !isInTheAir) {
    double latencySeconds = (LimelightHelpers.getLatency_Pipeline("")
                           + LimelightHelpers.getLatency_Capture("")) / 1000.0;
    addVisionMeasurement(visionPose, Timer.getFPGATimestamp() - latencySeconds);
}
```

The three filters:
1. **Zero pose** — `(0, 0, 0)` is the default return when no tag is visible
2. **Field boundary** — the FRC field is 16.54m × 8.21m; anything outside is impossible
3. **Z-axis check** — if the pose puts the robot in the air, it's wrong

> The field dimensions above are for the 2024 season. Check the game manual for the current year's field size.

In Step 7, you'll see how these filters are structured in our actual robot code. For now, implement them however makes sense to you.

---

## Pose Ambiguity

Even when the filters pass, vision can still return the wrong pose. This is called **ambiguity**.

<img src="https://docs.wpilib.org/en/stable/_images/planar_ambiguity1_base.png" alt="drawing" width="400"/>
<img src="https://docs.wpilib.org/en/stable/_images/planar_ambiguity1.png" alt="drawing" width="400"/>

A single 2D image of a tag can map to two completely different 3D positions — the tag looks identical from both viewpoints. The system doesn't know which one is right, so it sometimes picks the wrong one. In AdvantageScope this shows up as the vision pose rapidly flipping between two locations.

### Fixing Ambiguity

#### Option 1: Camera Angle (Most Effective)

Mounting cameras at **oblique (angled) positions** relative to the tags — rather than looking straight at them — significantly reduces ambiguity, because the two possible 3D interpretations diverge more when viewed from an angle.

> **Talk to your design team early.** Good communication with designers is critical. If the cameras end up in a bad spot mechanically, there's often little software can do to fix it. This applies to other mechanical decisions too (chain runs, backlash, etc.).

#### Option 2: Multitag / MT1

If two tags are visible in the same frame, the vision system can use both together to resolve ambiguity much more reliably — having two reference points essentially eliminates the ambiguous case.

- On Limelight this is called **MegaTag 1 (MT1)** and is enabled by default
- On PhotonVision this is called **Multitag** and needs to be enabled in the web UI and in code

#### Option 3: Gyro Correction / MT2

The gyro can be used to choose the correct interpretation when vision is ambiguous. You tell the system "I know I'm facing roughly this direction" and it picks whichever of the two ambiguous poses matches that heading.

- On Limelight this is called **MegaTag 2 (MT2)**

> **Warning:** MT2 produces very stable but potentially very wrong poses if your gyro angle is off. Make sure your gyro is reading correctly before enabling it — a drifted or flipped gyro will confidently lock onto the wrong pose.

---

## Checklist Before Moving On

- [ ] `LimelightHelpers.java` is in your project
- [ ] Vision pose is logged and appears in AdvantageScope when pointing at a tag
- [ ] `DifferentialDrivePoseEstimator` replaced `DifferentialDriveOdometry`
- [ ] Trackwidth constant is measured and added to `DrivetrainConstants`
- [ ] `update()` is still called every loop
- [ ] Vision measurements use the latency-corrected timestamp
- [ ] Zero pose, field boundary, and Z-axis filters are all implemented
- [ ] Odometry pose visibly corrects toward vision pose when tags are visible

---

**Tell a lead when you're done so they can verify your pose looks correct on the field.**
