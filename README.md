<h1 align="center">Finding security vulnerabilities with CodeQL</h1>
<h5 align="center">@adityasharad and @lcartey</h3>

<p align="center">
  <a href="#mega-prerequisites">Prerequisites</a> •  
  <a href="#books-resources">Resources</a>
</p>

> CodeQL is GitHub's expressive language and engine for code analysis, which allows you to explore source code to find bugs and security vulnerabilities. During these beginner-friendly workshops, you will learn to write queries in CodeQL and find known security vulnerabilities in open-source Java and JavaScript projects.

> There are two workshops on this topic. Both will cover the basics of writing queries in CodeQL. The first will focus on Java, and the second will focus on JavaScript.

## Workshop materials

Please complete the **Prerequisites** section (below) before the workshop.
The following links contain the content that will be covered during the workshop:
1. Thursday May 7 / 7:00am PDT: [Finding security vulnerabilities in Java with CodeQL](/java.md)
1. Thursday May 7 / 9:30am PDT: [Finding security vulnerabilities in JavaScript with CodeQL](/javascript.md)

## :mega: Prerequisites
- Install [Visual Studio Code](https://code.visualstudio.com/).
- Install the [CodeQL extension for Visual Studio Code](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html).
- You do _not_ need to install the CodeQL CLI: the extension will handle this for you.
- Set up the [CodeQL starter workspace](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html#using-the-starter-workspace).
  - **Important:** Don't forget to use `git clone --recursive` or `git submodule update --init --remote` to update the submodules when you clone this repository. This allows you to obtain the standard CodeQL query libraries.
  - Open the starter workspace in Visual Studio Code: **File** > **Open Workspace** > Browse to `vscode-codeql-starter/vscode-codeql-starter.code-workspace` in your checkout of the starter workspace.
- Download and add the CodeQL database to be used in the workshop:
  - If you are attending **Finding security vulnerabilities in Java with CodeQL**, please download [this CodeQL database](https://github.com/githubsatelliteworkshops/codeql/releases/download/v1.0/apache_struts_cve_2017_9805.zip).
  - If you are attending **Finding security vulnerabilities in JavaScript with CodeQL**, please download [this CodeQL database](https://github.com/githubsatelliteworkshops/codeql/releases/download/v1.0/esbena_bootstrap-pre-27047_javascript.zip)
  - Unzip the database.
  - Import the unzipped database into Visual Studio Code:
    - Click the CodeQL icon in the left sidebar.
    - Place your mouse over **Databases**, and click the `+` sign that appears on the right.
    - Choose the unzipped database directory on your filesystem.

## :books: Resources
- [CodeQL docs](https://codeql.github.com/docs/)
- [CodeQL for Java](https://codeql.github.com/docs/codeql-language-guides/codeql-for-java/)
- [CodeQL for JavaScript](https://codeql.github.com/docs/codeql-language-guides/codeql-for-javascript/)
- [CodeQL for Visual Studio Code](https://codeql.github.com/docs/codeql-for-visual-studio-code/)
- More about CodeQL on [GitHub Security Lab](https://securitylab.github.com/get-involved/)
- CodeQL on [GitHub Learning Lab](https://lab.github.com/githubtraining/codeql-u-boot-challenge-(cc++))
