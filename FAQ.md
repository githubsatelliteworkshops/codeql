# Frequently Asked Questions - CodeQL :artificial_satellite: workshops

## General
- **Will the slides be available?**
  - Yes, [here](https://github.com/githubsatelliteworkshops/codeql/blob/master/satellite-2020-workshops-codeql.pdf)
  
## CodeQL setup 
- **I’m getting `could not resolve module java` and queries don’t seem to be running… did I miss something obvious in setting this up?**
  - Make sure to get all sub-modules: `git clone --recursive https://github.com/github/vscode-codeql-starter/`
  - You might have an old version of the `codeql cli` installed in your path. Delete that and let the vscode extension install it

## CodeQL

- **It is possible to create custom code ql queries that run as part of CI/CD?**
  - Yes. For open-source projects you can configure the CodeQL GitHub Action to include custom queries you have added to your repository. For closed-source/enterprise code, you can do something similar once you have a license. The enterprise deployment of GitHub Advanced Security allows custom queries to be added and can be integrated into developer workflows.
- **Can CodeQL queries be run on the output of binary tools, such as LLVM or IDA, rather than on source code?**
  - Usually no. CodeQL databases are produced by extracting the source code during the build process - the CLI listens to the compiler and processes all source code that is compiled and built.
- **Is there human readable documentation outside VSCode where one can browse the available API (methods, class hiearchy etc)?**
  - The queries and standard libraries are open-sourced at http://github.com/codeql, and the documentation is available at https://help.semmle.com/QL/learn-ql/ and https://help.semmle.com/QL/ql-libraries.html.
- **Is there a repo/report/archive somewhere of you all running CodeQL and various vulnerabilities against a large number of open source repos already, or have you just run it on a few?**
  - https://securitylab.github.com has comprehensive information on vulnerabilities discovered on OSS with CodeQL.
  - https://LGTM.com runs CodeQL analysis for free on over 130k open-source repos.
  - This scanning will now be enabled directly on the GitHub.com platform for our users, via the Code Scanning Action. 
- **Is it possible to analyze the dependencies as dependencies? Namely to identify all code that has a specific dependency?**
  - In general it is possible to identify dependencies. The exact mechanism depends on the language being analysed. For example, we have the content of Maven POM files in Java databases, and package.json in JavaScript databases, and you can query those to find out what your code depends on. Identifying specifically which code uses those dependencies is more involved, though I think mostly possible. From a security scanning point of view, there are some other complementary GitHub tools (dependency graph, dependency insights) that give you an overview of this information on your repositories.
  - For the GitHub security feature: https://help.github.com/en/github/visualizing-repository-data-with-graphs/listing-the-packages-that-a-repository-depends-on
  - For the CodeQL Java library that lets you examine POM files: https://help.semmle.com/qldoc/java/semmle/code/xml/MavenPom.qll/type.MavenPom$Dependency.html or https://github.com/github/codeql/blob/master/java/ql/src/semmle/code/xml/MavenPom.qll.
- **Preference between `getName() = string`/`hasName(string)` ?**
  - Both are available for convenience. `hasName` with a specific string is shorter, but `getName` allows you to easily continue the condition, say, if you want to restrict the name with a regex like so: `.getName().regexpMatch(...)`.
- **What's the best way to extend a backend library to identify a new source of untrusted user input (in order to hopefully benefit from all the existing codeql queries)?**
  - The main out-of-the-box definition of untrusted input in the java QL libraries is called `RemoteFlowSource` and it is defined in `FlowSources.qll`.  This class allows you to extend it with more cases, by following the pattern in that file.
  - Custom extensions can be conveniently put in the file `Customizations.qll` where they'll be visible by all queries.
- **CodeQL has few "common" concepts, but are all differently named.  Makes the learning curve higher (for example Java `IfStmt/Block/getNumStmts` vs JavaScript `IfStmt/BasicBlock/getNumLines`).  Wish there was a higher level of abstraction so that queries were a bit more portable**
   - Actually, in this case JavaScript does have the same classes as Java (both `Block` and `getNumStmts()`).
   - One challenge we have is where different languages have standard names for things that are different. For example, the Java language spec defines a "call" as a "method access", which is why the class name is `MethodAccess`.
   - The other problem is that the concept may not be identical. In particular, a JavaScript `CallExpr` is quite different from a Java method call, because the target of the call can be defined dynamically.
 
