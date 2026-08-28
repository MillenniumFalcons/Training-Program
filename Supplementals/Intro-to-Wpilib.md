# Introduction to WPILIB
This guide will teach you how to install WPILIB VS code and create your first WPILib Project.  

## ***WPILib***
### What is WPILib?
<!---![](WPILibLogo.png)--->
<img src="images\WPILibLogo.png" alt="drawing" width="200"/>

WPiLib is code library developed by Worcester Polytechnic Institute (WPI) built for the ***FIRST Robotics Competition (FRC).*** This library contains helpful classes starting from direct interaction with motors, pneumatics, LEDs all the way to advanced path planning. Officially supported languages are Java and C/C++. ([Read More](https://docs.wpilib.org/en/stable/docs/software/what-is-wpilib.html#:~:text=The%20WPI%20Robotics%20Library%20(WPILib,and%20used%20by%20other%20software.)))

<!---![](WPILoc.jpeg)--->
<img src="images\WPILoc.jpeg" alt="drawing" width="200"/>

WPILib is the basis of FRC Programming. All other programs and libraries will built around WPILib. Teams start off with WPILib and build upon this central library when they become more advanced. 

WPILib usually has yearly releases parallel to yearly FRC game releases where the library of code is updated to fit new hardware and new features are added. 

### How to Install WPILib?
Please follow [this guide](https://docs.wpilib.org/en/stable/docs/zero-to-robot/step-2/wpilib-setup.html) to install WPILib.



## ***Project Creation***

- Hit `ctrl + p` (`cmd + shift + p` on mac) to open up the command palette
- Type `Create a new project` and hit enter

<img src="images\newnewproject.png" alt="drawing" width="1000"/>

***right click on image and click open image in new tab to view a bigger version of what to fill in***

- Fill out the Project Creator so it looks something like above.
	- ***make sure to chose `template -> java -> Command Robot Skeleton (advanced)` for the selector that's circled in the picture***
	- Chose which ever folder you want your project to be made in
	- The project name should indicate what year it is, the project should also be in pascal case. (e.g. `YEAR-SomeName`)
	- Type 3647 for the team number. You will not be able to deploy code to the robot if your have the wrong team number. 
- Click `Generate Project`. After this, a terminal should pop up at the bottom of your screen. 
> [!NOTE]  
> YEAR should be the current year (e.g. `2024-SomeName`).

***In about 35 seconds, your terminal sound read `BUILD SUCCESSFUL`***

### ***File Structure Adjustment***
***Current Default File Structure***
```
java
└── frc
    └── robot
        ├── commands
        │   └── ExampleCommand.java
        ├── Constants.java
        ├── Main.java
        ├── RobotContainer.java
        ├── Robot.java
        └── subsystems
            └── ExampleSubsystem.java
```

***Note: all folder names should be lower cased***

- Go into the java folder of your project through vscode or through file explorer
- Inside of the `java` folder, create a folder called `team 3647`
- Inside of the `team3647` folder you just made, create a folder called `frcYEAR` (e.g. frc2026 or whatever year it is) and `lib`
- Inside of `frcYEAR`, add
	- autonomous
	- commands
	- robot
	- subsystems
 	- constants 
- Move `Main.java`, `Robot.java`, and `RobotContainer.java` into team3647/frcYEAR/robot
- Delete the frc folder
- return null in getAutonomousCommand() in ```RobotContainer.java```

***Make appropriate edits to the package statements at the top of these files***

***Final Adjusted File Structure***
```
java
└── team3647
    ├── frcYEAR
    │   ├── autonomous
    │   ├── commands
    │   │   └── ExampleCommand.java
    │   ├── constants
    │   │   └── Constants.java
    │   ├── robot
    │   │   ├── Main.java
    │   │   ├── RobotContainer.java
    │   │   └── Robot.java
    │   └── subsystems
    │       └── ExampleSubsystem.java
    └── lib
```

### ***Modify build.gradle***
modify this line on build.gradle from:
```
def ROBOT_MAIN_CLASS = 'frc.robot.Main'
```
to 
```
def ROBOT_MAIN_CLASS = 'team3647.frcYEAR.robot.Main'
```

This is because the path to your `Main.java` file has changed after you adjust your file structure according to the instructions above. 

### ***Build Code***
- Hit `ctrl + shift + p` (`cmd + shift + p` on mac) to open up the command palette
- Type `build` and press enter for `WPILib: Build Robot Code`
- If all has been done correctly, the terminal should say `BUILD SUCCESSFUL` in about 30 seconds


## ***Authors***
- [Team 3647, Edward Sun](https://github.com/EdwardoSunny)












