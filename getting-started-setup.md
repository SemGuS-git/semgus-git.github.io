## Setup

*[Index](getting-started)*

This section covers the installation of the Semgus command-line interface, which provides access to our baseline enumerative solvers.

See [Tools and Libraries](tools) for other resources, such as the standalone parser CLI.

### Installing binaries

For convenience, the CLI is available as a pre-built binary for Windows, Mac, and Linux. [[Link]](https://github.com/SemGuS-git/Semgus-Operational/releases)

After downloading and unzipping the release for your platform, add the folder containing the executable to your `PATH`. You should then be able to run the CLI as described below.

### Compiling from source

The SemGuS parser, enumerative solvers, and CLI are built with .NET Core 6.0 using Visual Studio 2022. Older versions of Visual Studio may also be compatible.

- If Visual Studio is not available for your platform or you prefer to use other tools, additional configuration may be required to build the project.

To clone our source code from [GitHub](https://github.com/SemGuS-git/Semgus-Operational), run the following commands in your terminal:

```
git clone git@github.com:SemGuS-git/Semgus-Operational.git
cd Semgus-Operational
git submodule update -i -r
```

Open the solution file at `Semgus-Operational/Semgus-Interpreter/Semgus-Interpreter.sln` in Visual Studio, then build the project from VS.

You can then add the executable files to your PATH as discussed in the previous section.

### Validation

In your terminal, run `semgus-cli --help` to print the help string and verify that the CLI is installed.

*[Next](getting-started-cli)*