<h1 align="center">Finding security vulnerabilities with CodeQL</h1>
<h5 align="center">@adityasharad and @lcartey</h3>

<p align="center">
  <a href="#mega-prerequisites">Prerequisites</a> •  
  <a href="#books-resources">Resources</a>
</p>

> CodeQL is GitHub's expressive language and engine for code analysis, which allows you to explore source code to find bugs and security vulnerabilities. During these beginner-friendly workshops, you will learn to write queries in CodeQL and find known security vulnerabilities in open-source Java and JavaScript projects.

> There are two workshops on this topic. Both will cover the basics of writing queries in CodeQL. The first will focus on Java, and the second will focus on JavaScript.

## :watch: Workshop times
1. Thursday May 7 / 7:00am: Finding security vulnerabilities in Java with CodeQL
1. Thursday May 7 / 9:30am: Finding security vulnerabilities in JavaScript with CodeQL

## :mega: Prerequisites
- Install [Visual Studio Code](https://code.visualstudio.com/).
- Install the CodeQL extension for Visual Studio Code. [Full setup instructions are here.](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html)
- Set up the [CodeQL starter workspace](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html#using-the-starter-workspace).
  - **Important:** Don't forget to use `git clone --recursive` or `git submodule update --init --remote` to update the submodules when you clone this repository. This allows you to obtain the standard CodeQL query libraries.
  - Open the starter workspace in Visual Studio Code: **File** > **Open Workspace** > Browse to `vscode-codeql-starter/vscode-codeql-starter.code-workspace` in your checkout of the starter workspace.

## :books: Resources
- [Learning CodeQL](https://help.semmle.com/QL/learn-ql)
- [Learning CodeQL for Java](https://help.semmle.com/QL/learn-ql/java/ql-for-java.html)
- [Learning CodeQL for JavaScript](https://help.semmle.com/QL/learn-ql/javascript/ql-for-javascript.html)
- [Using the CodeQL extension for VS Code](https://help.semmle.com/codeql/codeql-for-vscode.html)
- More about CodeQL on [GitHub Security Lab](https://securitylab.github.com/tools/codeql)
- CodeQL on [GitHub Learning Lab](https://lab.github.com/githubtraining/codeql-u-boot-challenge-(cc++))
