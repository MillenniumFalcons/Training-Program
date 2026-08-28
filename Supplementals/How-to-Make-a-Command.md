## 1. The Slow Way
- make a class that implements command
- you need to override 4 methods: 
     1. `init()`
         - this will run once at the start of the command
     2. `execute()` 
         - this runs periodically while the command is active 
     3. `isFinished()`
         - this should return a Boolean, and tells the commandsheduler what it means for the command to "end"
     4. `end(Boolean interrupted)`
         - this defines what to do when the command ends. if you want different end behavior when the command is interrupted by another command, you can define that here using the parameter 
    once those methods are done, you simply make a new object with this class, and that object can be run like any other command. 

## 2. The in-between way: `FunctionalCommand`
`FunctionalCommand` lets you define the same 4 methods as the slow way (`initialize`, `execute`, `end`, `isFinished`), but as lambdas passed into a constructor instead of overriding methods in a class. This is useful when `Commands.run()` isn't enough (for example, when you need custom `isFinished()` or `end()` behavior) but you don't want to write a whole class.
- `new FunctionalCommand(Runnable onInit, Runnable onExecute, Consumer<Boolean> onEnd, BooleanSupplier isFinished, Subsystem... requirements)`
	- `onInit` runs once when the command starts (like `init()`)
	- `onExecute` runs periodically while the command is active (like `execute()`)
	- `onEnd` runs when the command ends, and receives a `Boolean` for whether it was interrupted (like `end(Boolean interrupted)`)
	- `isFinished` is checked periodically, and the command ends when it returns `true`
	- `requirements` is a list of subsystems for the requirements

```java
public Command turnWheel() {
    double[] target = new double[1];
    return new FunctionalCommand(
        () -> target[0] = ..., // initialize
        () -> drivetrain.setAngle(target[0]), // execute
        interrupted -> {}, // end
        () -> ..., // isFinished
        drivetrain
    );
}
```
> [!NOTE]
> Because the lambdas can't declare their own fields, this example stores `target` in a 1-element array so `initialize` can write to it and `execute` can read from it — lambdas can only capture variables that don't change after being assigned, but the contents of an array can still be mutated.

## 3. The fast way
You can also use the `Commands` helper class to create the commands you want quicky. *The `Commands` class contains helper methods that all return commands.* Here are some of the most common methods
- `Commands.run(Runnable toRun, Subsystem... requirements)`

    - the `Runnable` (in the form of a lambda) will run in the execute method of the returned command
    - the requirements is a list of subsystems for the requirements
- `Commands.Sequence(Command... commands)`
     - this takes a list of commands and runs them in sequence, one after the other
     - you can achieve the same affect by appending `.andThen(Command... commands)` to the end of a command
 - `Commands.parallel(Command... commands)` 
     - this takes a list of commands and runs them in parallel, at the same time 
	 - you can achieve the same effect by appending `.alongWith(Command... commands)` to the end of a command
To explore the other factories, you can look at [this page](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/wpilibj2/command/Commands.html) in the documentation
## 4. Running Commands With Triggers
 *This is the recommended way to run commands.* It involves binding the Commands to a `Trigger` object that will execute the Command when the condition is fulfilled
- Create a trigger using the constructor `new Trigger(BooleanSupplier condition)`
	- for the parameter, pass a lambda (or a function) that checks for the condition. (for example checking if a controller button is pressed) If you're unsure what a `BooleanSupplier` is, read [this guide](https://docs.wpilib.org/en/stable/docs/software/basic-programming/functions-as-data.html) 
- Bind a Command to your Trigger by using one of the methods
	- `.whileTrue(Command command)`
		- runs the parameter command while the condition of the trigger is true, stops it when the condition becomes false
	- `.onTrue(Command command)`
		- runs the parameter command once when the trigger is triggered, then stops it
	Both of these methods have opposites, `.whileFalse()` and `.onFalse()` 
- In our code, we use the `Joysticks` class, which creates Triggers for the buttons on an Xbox controller, so we can easily bind Commands to buttons 

> [!NOTE]
> In our code, we use the `Joysticks` class, which creates Triggers for the buttons on an Xbox controller, so we can easily bind Commands to buttons 



> [!IMPORTANT]
> The `Joysticks` class is not provided by Wpilib or a vendor, you need to copy it from an old robot's code

for more trigger methods, consult [this page](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/wpilibj2/command/button/Trigger.html)
