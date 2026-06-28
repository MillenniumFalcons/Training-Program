# Bank Account Mini Project — Example Solution

**BankAccount.java**
```java
public class BankAccount {
    String ownerName;
    double balance;
    int accountNumber;

    BankAccount(String ownerName, double balance, int accountNumber) {
        this.ownerName = ownerName;
        this.balance = balance;
        this.accountNumber = accountNumber;
    }

    void printInfo() {
        System.out.println("Owner: " + ownerName);
        System.out.println("Account #: " + accountNumber);
        System.out.println("Balance: $" + balance);
        System.out.println();
    }

    void deposit(double amount) {
        balance += amount;
    }

    void withdraw(double amount) {
        if (balance >= amount) {
            balance -= amount;
        } else {
            System.out.println("Warning: Not enough funds in " + ownerName + "'s account!");
        }
    }
}
```

**Main.java**
```java
public class Main {
    public static void main(String[] args) {
        BankAccount alice = new BankAccount("Alice", 500.0, 1001);
        BankAccount bob = new BankAccount("Bob", 200.0, 1002);

        alice.deposit(150.0);
        alice.withdraw(100.0);

        bob.deposit(50.0);
        bob.withdraw(300.0); // triggers warning

        alice.printInfo();
        bob.printInfo();

        if (alice.balance > bob.balance) {
            System.out.println(alice.ownerName + " has more money!");
        } else {
            System.out.println(bob.ownerName + " has more money!");
        }
    }
}
```

**Expected output:**
```
Warning: Not enough funds in Bob's account!
Owner: Alice
Account #: 1001
Balance: $550.0

Owner: Bob
Account #: 1002
Balance: $250.0

Alice has more money!
```
