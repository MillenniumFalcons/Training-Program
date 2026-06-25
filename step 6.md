# Step 5: Swerve

**Swerve Background:**
Swerve is the type of drivetrain where each wheel is free to rotate independently, which allows you to spin and drive at the same time. This is the drivetrain we use for most of our robots.


>[!NOTE]
>You should [create a new project](Supplementals/Intro-to-Wpilib.md#project-creation) for this exersize, and install the CTRE Phoenix vendor library

## CTRE Swerve API:
To actually run our swerve drivetrain, we’ll use the api that CTRE provides in their docs. However, you should learn how it works [under the hood](Supplementals/How%20to%20Structure%20a%20Custom%20Swerve%20Drivetrain.md) at some point, even if you don’t want to right now.

>[!IMPORTANT]
>Before you begin, please [read the CTRE Swerve API docs.](https://v6.docs.ctr-electronics.com/en/stable/docs/api-reference/mechanisms/swerve/swerve-overview.html) 
>For now, you don't need to worry much about actually making things, but make sure you know the theory and what everything does.
## Configuring a swerve drive
Thankfully, we don't acutally have to make the swerve drivetrain a swerve module configs manually. We don't even have to tune it that much!

All we have to do is run the Phoenix Tuner swerve builder routine. (read through all the subtabs in [this page](https://v6.docs.ctr-electronics.com/en/stable/docs/tuner/tuner-swerve/index.html), or just follow the directions in the phoenix tuner x app)

once you finish, click the "only generate TunerConstants" button, and save the file to the constants folder in you robot project

also, for safety, click the "save as json" button and save the json file somewhere.
## Running a swerve drive
1. We’ll start with a drivetrain class (called SwerveDrive) much like the minibot, but this one should extend `TunerSwerveDrivetrain`, created in step 1 
>[!NOTE]
> the `SwerveDrivetrain` class needs to know what motors and encoders you're using, so your drivetrain class should extend `TunerSwerveDrivetrain` that was generated in `TunerConstants`
- Keep in mind, the constructor should take all the necessary arguments so you can call the `super()` constructor (a `SwerveDrivetrainConstants` and 4 `SwerveModuleConstants`)

2. The process for setting a swerve drive is similar to the process of setting a TalonFX motor, except the motor is now your drivetrain class
	1. since your drivetrain class extends `SwerveDrivetrain`, you should be calling `super.setControl()` with a swerve Request parameter
3. Much like your drivetrain from the mini bot, the way we structure the swerve code is to use an "output" variable, and modify the variable whenever we call the drive function
	1. instead of a double, we'll use a `SwerveRequest`
#### Using the `SwerveRequests`
1. create a new generic swerve request, set it equal to a new idle swerve request `SwerveRequest name = new SwerveRequest.idle()`
2. then, we create swerve requests for each type of driving we want to do 
	- eg. one for field relative drive, one for robot relative drive, etc.
3. when you make your drive function, 
	1. modify the specific swerve requests for that type of drive (field centric for field relative drive, or robot centric for robot relative drive)
	2. set the generic control request equal to the specific control request `nameOfGenericRequest = nameOfSpecificRequest`


