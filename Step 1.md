# Step 1: Learn Java with W3Schools (if not already familiar with Java) 
*This should take at most 2 weeks.*

This is a link to [W3Schools](https://www.w3schools.com/java/default.asp) which is course for learning Java. At some point you will need to learn everything from the sections "Java Turorial", "Java Methods", and "Java Classes". 

Good news: you don't need to go through all of it. Throughout this curriculum, you should reference the W3Schools whenever you need to use a new concept or are unsure of something. 

If you already know a similar language like C++ or Python, you can just skim through this in a few minutes and move on to the next step.

---

## Part 1: The Basics

### 1 - Intro & Variables
Open [W3Schools](https://www.w3schools.com/java/default.asp) and read through the sections starting from Intro all the way through Variables. Then complete the Code Challenge at the end of variables. 

You do not need to read through every line, you can jump around as you desire. The goal is not to memorize all the rules for java, but rather to get a feel for the basics. You should be able to complete the Code Challenge without outside help.

Once you're done, if you're not bored yet, I recommend going through "Java Data Types" and "Java Operators" and doing the Code Challenges too. This is not strictly required at this point, but you will need to learn it eventually.

If you feel like you haven't gotten the hang of it, play around with java variables here: https://onecompiler.com/java until you're confident. You need to change the tab to I/O on the right side to see what the printed outputs.

### 2 - OOP: Classes & Methods
Skim through all the pages under "Java Methods" then do the Code Challenge. 

Skim through the pages starting from "Java OOP" through "Java this keyword" and do the Challenge.

It is okay if you don't fully grasp everything, it will make sense once you finish the mini project.

---

## Part 2: Mini Project
You now have a good idea of the basics of Java, so it's time to do a mini project. This will reinforce what you've already learned and introduce you to the rest of the main concepts you need to start programming a real robot.

You'll build a simple bank account simulator. Use this online compiler to write and run your code: [https://onecompiler.com/java](https://onecompiler.com/java) — it supports multiple files, so keep your classes in separate files. 

If you get stuck at any point, you can reference the [example solution](Supplementals/Bank-Account-Example.md) — but try to get as far as you can on your own first, you should only use this as a last resort! Try not to look at it until you've finished the mini project.

### Step 1 — Create a BankAccount class

Create a `BankAccount.java` file with a `BankAccount` class that has the following:
- **Fields:** `String ownerName`, `double balance`, `int accountNumber`
- **A constructor** that takes all three as parameters and sets the fields
- **A `printInfo()` method** that prints the owner name, account number, and current balance

Then create a separate `Main.java` file with a `main` method that creates one `BankAccount` and calls `printInfo()` on it to make sure it works.

### Step 2 — Add a deposit method

Add a `deposit(double amount)` method to your `BankAccount` class that adds `amount` to the balance. Test it in `main` by depositing some money and calling `printInfo()` again to confirm the balance changed.

### Step 3 — Add a withdraw method with a condition

Before writing the withdraw method, read the W3Schools page on [If...Else](https://www.w3schools.com/java/java_conditions.asp). This is also a good time to read through "Switch", "While Loop" and "For Loop" pages. Do all the Code Challenges.

An `if` statement lets your code make decisions. For example:
```java
if (balance >= amount) {
    // do the withdrawal
} else {
    // print a warning
}
```

Add a `withdraw(double amount)` method that:
- Subtracts `amount` from the balance **if** there are enough funds
- Prints a warning message and does nothing **if** there aren't enough funds

Test it by trying a normal withdrawal and then trying to withdraw more than the balance.

### Step 4 — Put it all together

In your `main` method:
- Create **two** `BankAccount` objects with different owners and starting balances
- Deposit and withdraw different amounts from each
- Intentionally try to overdraw one account to trigger the warning
- Use an `if/else` to check which account has a higher balance and print which owner "wins"

---

**Once you are done, or if you want to skip this step, come talk to one of the heads about the next steps.**

If none of us are available, your next step is to install WPILIB VS Code. Wpilib VS Code is the editor we use to program robots, and it has a bunch of integrations that make it easier to write and run code for robots. **To get started follow [this guide](Supplementals/Intro-to-Wpilib.md) to install Wpilib and make your first project.**

**Once you have done that, contact one of the leads for next steps.**

If none of us are available, **read through [Wpilib's command based programming guide](https://docs.wpilib.org/en/stable/docs/software/commandbased/index.html) and the [how to make a command guide](Supplementals/How-to-Make-a-Command.md), and get started on step 2.**

