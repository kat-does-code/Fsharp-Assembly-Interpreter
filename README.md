# Fsharp-Assembly-Interpreter
Archived project pre 2018 (est. 2019)

This project functions as an Assembly interpreter off of a text file containing Assembly instructions.

Capabilities:

- Aerithmatics (basic math)
    - `ADD`
    - `SUB`
    - `MUL`
    - `DIV`
- Control flow (function calls, conditional jumps)
    - `JMP`
    - `CMP`
    - `JGE` (and others)
    - `CALL`
- Data flow (moving data, functional stack)
    - `PUSH`
    - `POP`
    - `MOV`
- User defined interrupt table (system calls)
    - `INT 80` (`printf()`)
    - `INT 69` (`exit()`)
    - `INT 62` (`read()`)

# Table of contents
- [Story behind this project](#story-behind-this-project)
  - [Comparison to object oriented programming](#comparison-to-object-oriented-programming)
- [The deal with assembly](#the-deal-with-assembly)
  - [Building the fundamentals](#building-the-fundamentals)
    - [Numbers](#numbers)
    - [Variables](#variables)
    - [Functions](#functions)
      - [Expression types](#expression-types)

## Getting started
This project uses [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers). Make sure to have [docker](https://docker.com/) installed. 

To run this code, reopen this folder in its dev container, open a new bash terminal in the container and use the following commands.

```bash
dotnet run testAssembly.txt
```

Please note that if you get `error MSB3374`, you will need to correct the access rights to the project files using the following command.

```bash
sudo chown -R vscode:vscode .
```

If successfully run, you will be prompted to enter a number. The program will generate a list of numbers up to and including that number. An example of a successfuly run program is displayed below.

![Program ran successfully asking "Please enter a number to generate a list of numbers between 0 and your chosen number:", to which user entered "10". Program responded with "0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10,", "Do you want to start over? [y/n]", to which user entered "n". Program responded with "That's it for now. Exit code: 0"](doc/example_run.png)

# Story behind this project
I did this project to learn about how F#.NET works and what concepts the program language uses. F#.NET is fundamentally different from most programming languages I learned before. Most languages I had encountered were object oriented, whereas F#.NET is deemed a functional programming language. I learnt most of F# from https://fsharpforfunandprofit.com/.


## Comparison to object oriented programming
In object oriented programming languages, like C++, C#.NET, Java, Python, Javascript and the likes, the programmer stores data in structures called `objects`. `Objects` are defined by `classes`, which are essentially blueprints. The object is `instantiated` from a class and the data in these objects can be changed freely. Take for example a simple counter class in C#.NET.

```C#
public static class Counter {                           // Create new class called 'Counter'.
    private int value;                                  // Class has a variable that holds the number to which is counted.

    public Counter(){                                   // Whenever the class is newly constructed, it is returned (as Counter object),
        this.value = 0;                                 // with the value set to 0.
    }           

    public int GetCount() {                             // Also make sure we can check the count at any time without having to increase.
        return this.value;
    }

    public int Increase(){                              // Class has a function to increase the value by one every time it is called.
        this.value = this.value + 1;                    // It ups the variable by one and sets it in the object.
    }
}

public static void Main(String[] args){                 // At the start of our program where Main() is the entrypoint for our assumed console application,
    var counter1 = new Counter();                       // construct a new Counter object called "counter1",
    var countedValue = counter1.GetCount();             // and pass the counted value into a new variable called "newlyCountedValue".
                                                        // At this point, the value should be 0.
    
                                                        // Next, the value is printed to the console window.
    Console.WriteLine("The counted value is:", countedValue); 

    counter1.Increase();                                // Run the counter a few times for demonstration's sake
    counter1.Increase();                                // ...
    counter1.Increase();                                // ...
    counter1.Increase();                                // ...
    counter1.Increase();                                // ...
    counter1.Increase();                                // ...
    
    countedValue = counter1.GetCount()                  // Pass the counted value back to our local variable and print it to the console window.
                                                        // This should print 6.
    
    Console.WriteLine("The new counted value is:", countedValue); 
}
```

F#.NET is different in that by default, values (and values within objects) are immutable. As a variable is passed into a function, it is expected to change into something different. Ideally, you shouldn't see any mutations in variables at all in your code. The C#.NET code above could be written in F# as follows.

```F#
let Increase (value:int) =                              // Have a function that 
    let newValue = value + 1                            // increases the value given by 1
    newValue                                            // and return the new value to the caller

let countedValue1 = 0                                   // Create a new variable with value 0

let countedValue2 = countedValue1 |> Increase           // Increase the starting value (0) once; and
printf "The new counted value is: %i\n" countedValue2   // print it to console

                                                        // Increase the starting value (6) six times; and 
let countedValue3 = countedValue1 |> Increase |> Increase |> Increase |> Increase |> Increase |> Increase
printf "The new counted value is: %i\n" countedValue3   // print it to console
```

As you can see, types are not mandatory to be explicitly defined. Even the `(value:int)` type declaration is not mandated, but shown for clarification. Moreover, function piping is used often by F# programs to pipe an output state from one function into the next function as its argument. This does mean it is important to verify at the time of data input that the data is not corrupt.

# The deal with assembly
The easiest way to start working on an interpreter for a (made up) programming language is by defining what the computer does easiest which is plain math; addition, subtraction, division and multiplication being the simplest. Assembly has these operations clearly mapped out which is why I decided to use it albeit an interpreted form. Furthermore, the statefulness of assembly seemed ideal, a clear state before and after an operation is executed. However, I did cheat and not recreate input and output states; instead I used some state variables declared in [Functions.fs](src/Functions.fs).

## Building the fundamentals
Starting by defining functions `add`, `sub`, `mul`, and `div`, we learn about F#'s aerithmatic operations, which unsurprisingly are the same as most other languages I've come across. However, in assembly it is possible to add the contents of a register to the contents of another register, or add a constant value to a register. To make sure we account for both of these cases and none other ( it would be impossible to store a register or constant data in constant data ), we can use `pattern matching`, an epic concept that is now also available for C#.NET.

> :warning: Another difference with assembly and this interpreter is I'm **not** using any registers as things can be achieved with variables just the same, which is a lot easier. 

Here's where things get interesting as assembly spec declares the following rules (excluding registers):

1. `ADD` may be executed with a left operand (variable) and a right operand (variable), in which case the value in the right variable is added to the value in the left variable.
2. `ADD` may be executed with a left operand (variable) and a right operand (constant), in which case the value of the right constant is added to the value in the left variable.
3. In both cases, the result is stored in the left operand (variable).

Now we're dealing with the concepts of variables, constants, math and exception cases. 

### Numbers
First let's define what a number actually is. In some programming languages, there's a separation between integer (whole) and real (fractional) numbers. In our case there's only hassle if we separate the two, so let's define a number as either an integer or real, or in F# primitive data types: `int` or `float`. 

F# allows for multitype definition. The following is perfectly valid F# code which means a Number type is of either `int` or `float`. And we don't care too much about the specifics of using real or integer numbers for our little project.

```F#
type Number =
    |Integer of int
    |Real of float
```
__[Types.fs:22](src/Types.fs#22)__

Great! We can have numbers. Now all that's left is the rest. 

Another basic type that is used throughout programming language is the string, character array, text, what have you. We'll call our type `Text` as we like to call it what it is. The F# primitive data type for a text is a `string`. However we do not need to redefine this at this time.

### Variables
The concept of variables is relatively simple. A variable is defined by a name, a value and a type (inferred or explicit). For our project, we'll keep ourselves to text and numbers, thus we can define our value type as follows.

```F#
type ValType = 
    |Text of string
    |Number of Number
```
__[Types.fs:60](src/Types.fs#60)__

Now that we have defined the types we can use for values, both for constants and variables, let's create the latter ones! As mentioned before, a variable consists of a name and a value, and we'll infer the type of that value implicitly.

```F#
type Constant(value) = 
    let mutable value:ValType = value
    member this.Data
        with get() = value
        and set(newVal) = value <- newVal

type Variable(name, initValue) = 
    let mutable value:ValType = initValue
    member this.Value
        with get() = value
        and set(newVal:ValType) = value <- newVal
    member this.Name:string = name
```
__[Types.fs:94](src/Types.fs#94)__

### Functions and expressions
Now that we can access constants and variables, we can create their counterpart: functions. The building blocks of a function are relatively simple, they consist of a name, a returned type,  arguments (optionally) and a list of expressions. Now `expressions` is what the entirety of a program is made up of. It causes one state to evolve into the next by evaluating the expressions one by one sequentially. There are four distinct expression types we can define in our program and have a functional interpreter:

1. Variable declarations, e.g. `int a;`
2. Variable (re)assignments, e.g. `a = 2;`
3. Function declarations, e.g. `int add(int x, int y){ return x+y }`
4. Function calls, e.g. `add(a, 4)`

We can also define expressions by the number of operands required. Intel x86 assembly has expressions with zero to three operands.

- Zero-operand expressions take no arguments. Examples of these are [`RET`](https://www.felixcloutier.com/x86/ret.html) and [`NOP`](https://www.felixcloutier.com/x86/nop.html).
- One-operand expressions take one argument. Examples of these are [`INC`](https://www.felixcloutier.com/x86/inc.html) and [`JMP`](https://www.felixcloutier.com/x86/jmp.html).
- Two-operand expressions take two arguments. Examples of these are [`ADD`](https://www.felixcloutier.com/x86/add.html) and [`CMP`](https://www.felixcloutier.com/x86/cmp.html).

Furthermore we can tag these by type:

- Aerithmatic expressions use math, such as `ADD`, `SUB`, `DIV` and `MUL`.
- Control flow expressions are used to (possibly conditionally) jump around your program, such as `JMP`, `CMP` and `CALL`.
- Data expressions are used to move data around, such as `MOV`, `PUSH`, `POP`.

Definitions of these can be viewed in [Expressions.fs](src/Expressions.fs).
