# Sample Swerve Code

Reference implementations for Step 6. These match the structure described in the step — read the step first, then use these to check your work.

---

## SwerveDrive.java

```java
package team3647.frc2026.subsystems;

import org.littletonrobotics.junction.Logger;

import com.ctre.phoenix6.swerve.SwerveDrivetrainConstants;
import com.ctre.phoenix6.swerve.SwerveModuleConstants;
import com.ctre.phoenix6.swerve.SwerveRequest;
import com.ctre.phoenix6.swerve.SwerveDrivetrain.SwerveDriveState;

import edu.wpi.first.math.geometry.Pose2d;
import edu.wpi.first.math.kinematics.SwerveModuleState;

import team3647.frc2026.constants.TunerConstants;
import team3647.lib.PeriodicSubsystem;

public class SwerveDrive extends TunerConstants.TunerSwerveDrivetrain implements PeriodicSubsystem {

    public class PeriodicIO {
        public SwerveRequest masterRequest = new SwerveRequest.Idle();
        public SwerveRequest.FieldCentric fieldCentric = new SwerveRequest.FieldCentric();
        public SwerveRequest.RobotCentric robotCentric = new SwerveRequest.RobotCentric();

        public SwerveModuleState[] states = new SwerveModuleState[4];
        public SwerveModuleState[] target = new SwerveModuleState[4];
        public Pose2d pose = new Pose2d();
    }

    PeriodicIO periodicIO = new PeriodicIO();

    public SwerveDrive(
            SwerveDrivetrainConstants drivetrainConstants,
            SwerveModuleConstants<?, ?, ?>... modules) {
        super(drivetrainConstants, modules);
        registerTelemetry(this::setStates);
    }

    public void setStates(SwerveDriveState state) {
        periodicIO.states = state.ModuleStates;
        periodicIO.target = state.ModuleTargets;
        periodicIO.pose = state.Pose;
    }

    @Override
    public void readPeriodicInputs() {
        Logger.recordOutput("Drivetrain/Pose", periodicIO.pose);
        Logger.recordOutput("Drivetrain/States", periodicIO.states);
        Logger.recordOutput("Drivetrain/Targets", periodicIO.target);
    }

    @Override
    public void writePeriodicOutputs() {
        setControl(periodicIO.masterRequest);
    }

    @Override
    public void periodic() {
        readPeriodicInputs();
        writePeriodicOutputs();
    }

    public void driveFieldCentric(double vx, double vy, double rotationRate) {
        periodicIO.fieldCentric
            .withVelocityX(vx)
            .withVelocityY(vy)
            .withRotationalRate(rotationRate);
        periodicIO.masterRequest = periodicIO.fieldCentric;
    }

    public void driveRobotCentric(double vx, double vy, double rotationRate) {
        periodicIO.robotCentric
            .withVelocityX(vx)
            .withVelocityY(vy)
            .withRotationalRate(rotationRate);
        periodicIO.masterRequest = periodicIO.robotCentric;
    }

    public Pose2d getPose() {
        return periodicIO.pose;
    }

    @Override
    public String getName() {
        return "drivetrain";
    }
}
```

---

## DrivetrainCommands.java

```java
package team3647.frc2026.commands;

import java.util.function.DoubleSupplier;

import edu.wpi.first.units.measure.LinearVelocity;
import edu.wpi.first.wpilibj.DriverStation;
import edu.wpi.first.wpilibj.DriverStation.Alliance;
import edu.wpi.first.wpilibj2.command.Command;
import edu.wpi.first.wpilibj2.command.Commands;

import team3647.frc2026.constants.TunerConstants;
import team3647.frc2026.subsystems.SwerveDrive;

public class DrivetrainCommands {

    private final SwerveDrive swerveDrive;
    private final double maxSpeed = TunerConstants.kSpeedAt12Volts.baseUnitMagnitude();

    public DrivetrainCommands(SwerveDrive swerveDrive) {
        this.swerveDrive = swerveDrive;
    }

    // Field-relative drive — flips direction on Red alliance so "forward" always
    // matches the driver's perspective regardless of which side they're on.
    public Command fieldDrive(DoubleSupplier leftY, DoubleSupplier leftX, DoubleSupplier rightX) {
        return Commands.run(() -> {
            double invert = DriverStation.getAlliance().orElse(Alliance.Blue) == Alliance.Red ? -1 : 1;
            swerveDrive.driveFieldCentric(
                invert * maxSpeed * leftY.getAsDouble(),
                invert * maxSpeed * leftX.getAsDouble(),
                maxSpeed * rightX.getAsDouble()
            );
        }, swerveDrive);
    }

    // Robot-relative drive — useful for auto or precise adjustments.
    public Command robotDrive(DoubleSupplier leftY, DoubleSupplier leftX, DoubleSupplier rightX) {
        return Commands.run(() -> {
            swerveDrive.driveRobotCentric(
                maxSpeed * leftY.getAsDouble(),
                maxSpeed * leftX.getAsDouble(),
                maxSpeed * rightX.getAsDouble()
            );
        }, swerveDrive);
    }
}
```

---

## RobotContainer.java

```java
package team3647.frc2026.robot;

import edu.wpi.first.wpilibj2.command.CommandScheduler;

import team3647.frc2026.commands.DrivetrainCommands;
import team3647.frc2026.constants.TunerConstants;
import team3647.frc2026.subsystems.SwerveDrive;
import team3647.lib.inputs.Joysticks;

public class RobotContainer {

    final Joysticks driver = new Joysticks(0);

    final SwerveDrive swervedrive = new SwerveDrive(
        TunerConstants.DrivetrainConstants,
        TunerConstants.FrontLeft,
        TunerConstants.FrontRight,
        TunerConstants.BackLeft,
        TunerConstants.BackRight
    );

    final DrivetrainCommands swerveCommands = new DrivetrainCommands(swervedrive);

    public RobotContainer() {
        CommandScheduler.getInstance().registerSubsystem(swervedrive);
        configureBindings();
        configureDefaultCommands();
    }

    private void configureBindings() {
        // bind additional commands to buttons here
    }

    private void configureDefaultCommands() {
        swervedrive.setDefaultCommand(
            swerveCommands.fieldDrive(
                driver::getLeftStickY,
                driver::getLeftStickX,
                driver::getRightStickX
            )
        );
    }
}
```
