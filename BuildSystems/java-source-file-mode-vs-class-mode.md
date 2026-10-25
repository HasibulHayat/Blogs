# Java Source-File Mode vs. Compiled Class Mode

*A practical note on why `java Main` and `java Main.java` can behave
differently.*

## 1. The experiment

Suppose you start with this source file:

``` java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

Compile it:

``` bash
javac Main.java
```

Your directory now contains:

``` text
JavaTest/
├── Main.java
└── Main.class
```

The two files represent different things:

-   `Main.java` is your source code.
-   `Main.class` is the compiled class file containing Java bytecode and
    metadata.

Now change `Main.java` so it prints a different message, but do **not**
compile it again.

``` java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, Version 2!");
    }
}
```

Your source file and compiled class now represent different versions of
the program.

## 2. Why does `java Main` run the old version?

When you run:

``` bash
java Main
```

you are asking the Java launcher to launch the class named `Main`.

In ordinary class-launching mode, Java looks for the compiled class
using the classpath and runs it. It does not automatically recompile
`Main.java` just because that source file is newer.

If the existing `Main.class` was compiled before you changed the source,
it can still contain the old instructions. So the output can remain:

``` text
Hello, Java!
```

This teaches an important principle:

> **Changing a source file does not automatically change the compiled
> class file.**

To update the compiled class, compile the source again:

``` bash
javac Main.java
```

Then run it:

``` bash
java Main
```

Now the output should reflect the updated source.

## 3. Why does `java Main.java` work differently?

Modern Java supports **source-file mode**, introduced in Java 11.

When you run:

``` bash
java Main.java
```

you tell the Java launcher to use `Main.java` as a source file. The
launcher compiles that source and executes the resulting program for
you.

Conceptually:

``` text
java Main
    |
    v
Find and launch the compiled Main class
    |
    v
Execute the class file

-----------------------------------------

java Main.java
    |
    v
Read Main.java
    |
    v
Compile the source
    |
    v
Execute the resulting program
```

Source-file mode does not mean the JVM directly executes Java source
text. Compilation still occurs; the launcher simply handles it for you.

This mode is convenient for small programs, experiments, teaching, and
scripts where you do not want to run `javac` as a separate command.

## 4. Why might `java Main.java` print two lines?

If the output is:

``` text
Hello, Java!
Hello, Version 2!
```

then the program executed in source-file mode produced two lines of
output. A common explanation is that the current source file contains
two print statements, for example:

``` java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
        System.out.println("Hello, Version 2!");
    }
}
```

If the source contains only one print statement, that output would not
be expected from that source alone. The right way to investigate would
be to inspect the source file and, if useful, inspect the existing
compiled class. But if you already understand the distinction, there is
no need to investigate further.

The key point is that `java Main.java` uses the source file as its
input, while `java Main` launches the compiled class found through the
classpath.

## 5. Comparing the commands

  -----------------------------------------------------------------------
  Command                             Meaning
  ----------------------------------- -----------------------------------
  `javac Main.java`                   Explicitly compile the source file
                                      into class file(s).

  `java Main`                         Launch the already-compiled class
                                      named `Main`.

  `java Main.java`                    Use source-file mode to compile and
                                      run the source file.

  `javap -c Main`                     Disassemble a compiled class and
                                      display its bytecode instructions.
  -----------------------------------------------------------------------

### A note about `javap`

`javap` is a JDK tool that can inspect compiled class files. The `-c`
option asks it to show the bytecode instructions for methods. This is
useful when you want to see what the compiler produced, although you do
not need it for everyday Java development.

## 6. Does source-file mode create or update `Main.class`?

Do not rely on source-file mode to update the `Main.class` file in your
project directory. Its purpose is to compile and run a source file
conveniently, not to act as a replacement for a deliberate project
build. Its compilation behavior and output handling differ from an
explicit `javac` invocation.

If you want a class file you can manage and run in the usual way,
compile explicitly:

``` bash
javac Main.java
java Main
```

For a multi-file project, explicit compilation also makes it easier to
control the output directory, packages, and classpath.

## 7. Why this matters for build systems

Imagine editing one file in a project containing hundreds of Java
classes.

The source files represent what the developer intends the program to be.
The class files represent a compiled result from some point in time.
They can become out of sync if the code changes and compilation is not
repeated.

A build system such as Maven or Gradle coordinates the process of
compiling source files, handling dependencies, running tests, copying
resources, and packaging the application.

For example:

``` bash
./gradlew build
```

Gradle can determine which configured build tasks need to run and
produce the project's build artifacts. The exact tasks and outputs
depend on the project configuration.

This is one reason build systems are valuable: they make the
relationship between source code and generated artifacts repeatable and
manageable.

## 8. The mental model

Keep these three ideas separate:

1.  **Source code:** the `.java` files you edit.
2.  **Compiled classes:** the `.class` files produced by compilation.
3.  **Execution:** the Java runtime loading and running compiled code.

The commands reflect those differences:

-   `javac Main.java` --- compile explicitly.
-   `java Main` --- launch a compiled class.
-   `java Main.java` --- let the launcher compile and run a source file.

For understanding build systems, the most important lesson is that
**source code and compiled output are separate artifacts, and some
process must keep the output up to date**.

------------------------------------------------------------------------

## Next lesson

Next, create a small application with two Java source files. Have one
class call a method from the other, then compile and run them manually.
That will introduce class dependencies and lead naturally to packages,
output directories, and the classpath.
