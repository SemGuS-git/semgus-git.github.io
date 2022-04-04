## SemGuS Tools and Libraries
We provide libraries for parsing SemGuS problems, as well as a command-line tool for verifying benchmarks and converting into a JSON intermediate representation.

### The SemGuS Parser
[View the SemGuS Parser on GitHub](https://github.com/SemGuS-git/Semgus-Parser).

The SemGuS Parser is a command-line tool used for checking SemGuS problems for correctness, as well as translating problem files into other representations.
Currently, a JSON representation is available. Download the tool from the [GitHub Releases page](https://github.com/SemGuS-git/Semgus-Parser/releases), or
install from NuGet via the .NET 6 SDK :

```
dotnet tool install --global Semgus.Parser.Tool
```

Usage and other instructions are available on GitHub. Issues and feedback are also welcome.

### The SemGuS Parser C# Library
[View the SemGuS Parser C# Library on GitHub](https://github.com/SemGuS-git/Semgus-Parser).

[View the SemGuS Parser C# Library on NuGet](https://www.nuget.org/packages/Semgus.Parser).

The C# library powering the SemGuS Parser is available for general use. Find the `Semgus.Parser` package in NuGet. Full documentation is in progress, 
but in the meantime, either make a discussion post on GitHub or reach out to [the SemGuS team](mailto:semgus@office365.wisc.edu) with any questions.

### The SemGuS Java Consumer Library
[View the SemGuS Java Consumer Library on GitHub](https://github.com/SemGuS-git/Semgus-Java).

[View the SemGuS Java Consumer Library on JitPack](https://jitpack.io/#SemGuS-git/Semgus-Java).

A Java library that consumes the JSON output of the SemGuS Parser is available. Visit the GitHub page for more information.
