# Java Practice Notes: From `touch Main.java` to Separating Source and Compiled Files

*A hands-on guide to creating Java files, using multiple classes,
understanding packages and classpaths, and organizing source code
separately from compiled output.*

------------------------------------------------------------------------

## 1. Create a Java source file with `touch`

On Linux, `touch` creates an empty file if the file does not already
exist.

For example:

``` bash
mkdir -p ~/JavaTest
cd ~/JavaTest
touch Main.java
ls
```

Expected listing:

``` text
Main.java
```

If the file already exists, `touch Main.java` updates its modification
timestamp; it does not erase the file's contents.

Open it in a terminal editor:

``` bash
nano Main.java
```

Add this code:

``` java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

Save with `Ctrl+O`, press `Enter`, then exit with `Ctrl+X`.

## 2. Compile and run one Java class manually

Compile the source:

``` bash
javac Main.java
```

List the files:

``` bash
ls
```

You should now see:

``` text
Main.java
Main.class
```

The files have different purposes:

-   `Main.java` is the source code you write.
-   `Main.class` is the compiled class file containing Java bytecode and
    metadata.

Run the compiled class:

``` bash
java Main
```

Expected output:

``` text
Hello, Java!
```

Notice that the command uses `Main`, not `Main.class`.

### Source-file mode

Modern Java also supports:

``` bash
java Main.java
```

This is source-file mode. The Java launcher compiles and runs the source
file for you. It is convenient for small experiments, but for learning
conventional project builds, keep using `javac` and `java` separately so
you can see the compilation step explicitly.

The two commands are different:

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `javac Main.java`                   Explicitly compile source code.

  `java Main`                         Launch the compiled class named
                                      `Main`.

  `java Main.java`                    Compile and run a source file using
                                      source-file mode.
  -----------------------------------------------------------------------

## 3. Add a second class

Create a second source file:

``` bash
touch Greetings.java
nano Greetings.java
```

Add:

``` java
public class Greetings {
    public static void sayHello() {
        System.out.println("Hello from Greetings!");
    }
}
```

Update `Main.java`:

``` java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
        Greetings.sayHello();
    }
}
```

Because both classes are in the default package, no `import` statement
is required.

Compile both files:

``` bash
javac Main.java Greetings.java
```

Run the application:

``` bash
java Main
```

Expected output:

``` text
Hello, Java!
Hello from Greetings!
```

### Can `javac Main.java` compile the other class automatically?

Often, yes. With the default source-path behavior, the compiler can
discover a referenced source file in the same directory and compile it
when needed:

``` bash
javac Main.java
```

This depends on the compiler options and where the source files are
located. Explicitly naming both files is clearer while learning.

## 4. Source files and compiled files are separate

Suppose you edit `Main.java` after compiling it. The existing
`Main.class` does not automatically update.

For example, change the message to:

``` java
System.out.println("Version 2!");
```

If you then run:

``` bash
java Main
```

you may still see the output from the previous compiled version, because
you have not recompiled the source.

Recompile:

``` bash
javac Main.java
java Main
```

Now the class file reflects the updated source.

This is a foundational idea:

> **Source code and compiled output are separate artifacts. Changing one
> does not automatically change the other.**

## 5. What is a classpath?

The classpath tells Java where to find compiled classes and resources
needed by an application.

Consider this structure:

``` text
JavaTest/
├── Main.java
├── Main.class
└── new/
    ├── Greetings.java
    └── Greetings.class
```

Suppose `Main` calls `Greetings.sayHello()`, but `Greetings.class` is in
`new/`.

Running:

``` bash
java Main
```

may fail with `NoClassDefFoundError`, because Java's default classpath
for this classpath-based launch is the current directory, not every
subdirectory beneath it.

On Linux and macOS, you can explicitly add both directories:

``` bash
java -cp ".:new" Main
```

Here:

-   `-cp` means classpath.
-   `.` is the current directory.
-   `new` is the directory containing `Greetings.class`.
-   `:` separates classpath entries on Linux and macOS.

On Windows, classpath entries are typically separated with `;` instead.

The compiler also needs to find referenced classes. For example:

``` bash
javac -cp ".:new" Main.java
```

This provides both locations when the compiler resolves class
references. The runtime still needs its own classpath when launching the
application.

**Important:** Java does not automatically search every subdirectory as
a separate classpath root.

## 6. Packages and imports

A Java package is a namespace for organizing related classes.

A package declaration such as:

``` java
package greetings;
```

belongs at the beginning of `Greetings.java`, before the class
declaration.

Example source:

``` java
package greetings;

public class Greetings {
    public static void sayHello() {
        System.out.println("Hello from a named package!");
    }
}
```

The source directory conventionally mirrors the package name:

``` text
JavaTest/
├── Main.java
└── greetings/
    └── Greetings.java
```

Then `Main.java` can import the class:

``` java
import greetings.Greetings;

public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
        Greetings.sayHello();
    }
}
```

An `import` statement lets you refer to `Greetings` by its simple name.
It does **not** download the class, compile it, or add a directory to
the runtime classpath. The class must still be available to the compiler
and runtime.

### Why not name the package `new`?

`new` is a reserved Java keyword, used for object creation. It cannot be
used as a package identifier.

Prefer lowercase package names such as:

-   `greetings`
-   `com.example`
-   `com.company.application`

## 7. Compile into a separate output directory

A clean project keeps source code separate from generated class files.

Create this structure:

``` text
JavaTest/
├── src/
│   ├── Main.java
│   └── greetings/
│       └── Greetings.java
└── out/
```

If you are reorganizing an existing practice project, first move the
source files into the intended structure and remove stale `.class` files
from the old locations. Do not delete files unless you have confirmed
you are in the correct project directory.

For example, from `~/JavaTest`:

``` bash
mkdir -p src/greetings out
```

Move the source files if they currently live in the project root and the
`greetings` directory:

``` bash
mv Main.java src/
mv greetings/Greetings.java src/greetings/
```

Make sure `src/greetings/Greetings.java` begins with:

``` java
package greetings;
```

And make sure `src/Main.java` imports it:

``` java
import greetings.Greetings;
```

Compile both source files into `out/`:

``` bash
javac -d out src/Main.java src/greetings/Greetings.java
```

The `-d out` option tells the compiler to place generated class files
under the `out` directory, creating package subdirectories as needed.

Inspect the output:

``` bash
find out -type f
```

Expected files:

``` text
out/Main.class
out/greetings/Greetings.class
```

Run the application:

``` bash
java -cp out Main
```

Expected output:

``` text
Hello, Java!
Hello from a named package!
```

Here, `out` is the classpath root. Java looks for the main class at
`out/Main.class` and the packaged class at
`out/greetings/Greetings.class`.

## 8. What each command does

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `touch Main.java`                   Create an empty source file if it
                                      does not exist.

  `nano Main.java`                    Edit a source file in a terminal.

  `javac Main.java`                   Compile Java source code.

  `javac Main.java Greetings.java`    Compile multiple source files
                                      explicitly.

  `javac -cp ".:new" Main.java`       Compile while making the current
                                      directory and `new` available for
                                      class resolution on Linux/macOS.

  `javac -d out ...`                  Write generated class files under
                                      `out/`.

  `java Main`                         Launch a compiled class named
                                      `Main`.

  `java -cp out Main`                 Launch `Main` with `out` as the
                                      classpath root.

  `java Main.java`                    Compile and run a source file using
                                      source-file mode.

  `find out -type f`                  List files beneath the output
                                      directory on Linux.
  -----------------------------------------------------------------------

## 9. Common errors and what they mean

### `cannot find symbol`

The compiler cannot resolve a name such as a class or method. Possible
causes include a spelling error, a missing import, an unavailable
dependency, or an incorrect source/class path.

### `ClassNotFoundException`

A class loader could not find a class it was asked to load. Check the
runtime classpath and the class's package name and location.

### `NoClassDefFoundError`

The JVM needed a class definition during execution but could not load
it. One common cause is that a class available at compile time is
missing from the runtime classpath.

### Source code and output disagree

You edited a `.java` file but did not recompile it. Run `javac` again or
use your build system to rebuild the project.

### `package new does not exist` or a syntax error near `new`

`new` is a reserved keyword, so it cannot be used as a package name.
Choose a valid name such as `greetings`.

## 10. Why this matters for Maven and Gradle

For a tiny application, manually compiling source files is manageable.
For a real application, you might have hundreds of classes, many
packages, external libraries, tests, resource files, and multiple build
steps.

Maven and Gradle automate and coordinate these tasks:

-   Resolve external dependencies.
-   Compile source files with the appropriate settings.
-   Place compiled output in conventional directories.
-   Copy resources.
-   Compile and run tests.
-   Package the application into a JAR or another artifact.

A typical Gradle Java project uses directories such as:

``` text
src/main/java/     Java source code
src/main/resources/ Application resources
src/test/java/     Test source code
build/classes/     Compiled classes
build/libs/        Packaged artifacts, such as JARs
```

The exact structure depends on the build configuration, but the
principle is the same as our manual `src/` and `out/` example.

A build system does not replace Java compilation or the JVM. It
automates the repetitive work required to turn source code, resources,
and dependencies into a build artifact.

## 11. Practice checklist

-   [ ] Create a Java source file with `touch`.
-   [ ] Compile it with `javac`.
-   [ ] Run the compiled class with `java`.
-   [ ] Create a second class and call one of its methods.
-   [ ] Observe what happens when a source file changes without
    recompiling.
-   [ ] Use the classpath to make a class in another directory
    available.
-   [ ] Declare a named package and import its class.
-   [ ] Compile into a separate directory using `javac -d out`.
-   [ ] Run using `java -cp out Main`.

## The mental model

Keep these concepts separate:

1.  **Source file:** the `.java` file you write.
2.  **Package:** the namespace a Java type belongs to.
3.  **Import:** a way to refer to a type by its simple name.
4.  **Compilation:** converting source code into class files.
5.  **Output directory:** where the compiler writes generated class
    files.
6.  **Classpath:** where the compiler or runtime searches for classes
    and resources.
7.  **Execution:** the JVM loading and running the compiled program.

Once these concepts are clear, the next step is to introduce external
JAR dependencies and then use Maven or Gradle to manage compilation,
dependencies, testing, and packaging.
