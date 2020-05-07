# CodeQL workshop for JavaScript: Finding unsafe calls to the jQuery `$` function

- Analyzed language: JavaScript
- Difficulty level: 200

## Overview

 - [Problem statement](#problemstatement)
 - [Setup instructions](#setupinstructions)
 - [Documentation links](#documentationlinks)
 - [Workshop](#workshop)
   - [Section 1: Finding calls to the jQuery `$` function](#section1)
   - [Section 2: Finding jQuery plugin options](#section2)
   - [Section 3: Finding XSS vulnerabilities](#section3)

## Problem statement <a id="problemstatement"></a>

jQuery is an extremely popular, but old, open source JavaScript library designed to simplify things like HTML document traversal and manipulation, event handling, animation, and Ajax. The jQuery library supports modular plugins to extend its capabilities. Bootstrap is another popular JavaScript library, which has used jQuery's plugin mechanism extensively. However, the jQuery plugins inside Bootstrap used to be implemented in an unsafe way that could make the users of Bootstrap vulnerable to cross-site scripting (XSS) attacks. This is when an attacker uses a web application to send malicious code, generally in the form of a browser side script, to a different end user.

Four such vulnerabilities in Bootstrap jQuery plugins were fixed in [this pull request](https://github.com/twbs/bootstrap/pull/27047), and each was assigned a CVE.

The core mistake in these plugins was the use of the omnipotent jQuery `$` function to process the options that were passed to the plugin. For example, consider the following snippet from a simple jQuery plugin:

```javascript
let text = $(options.textSrcSelector).text();
```

This plugin decides which HTML element to read text from by evaluating `options.textSrcSelector` as a CSS-selector, or that is the intention at least. The problem in this example is that `$(options.textSrcSelector)` will execute JavaScript code instead if the value of `options.textSrcSelector` is a string like `"<img src=x onerror=alert(1)>".` The values in `options` cannot always be trusted.

In security terminology, jQuery plugin options are a **source** of user input, and the argument of `$` is an XSS **sink**.

The pull request linked above shows one approach to making such plugins safer: use a more specialized, safer function like `$(document).find` instead of `$`.
```javascript
let text = $(document).find(options.textSrcSelector).text();
```

In this challenge, we will use CodeQL to analyze the source code of Bootstrap, taken from before these vulnerabilities were patched, and identify the vulnerabilities.

## Setup instructions for Visual Studio Code <a id="setupinstructions"></a>

To take part in the workshop you will need to follow these steps to get the CodeQL development environment set up:

1. Install the Visual Studio Code IDE.
1. Download and install the [CodeQL extension for Visual Studio Code](https://help.semmle.com/codeql/codeql-for-vscode.html). Full setup instructions are [here](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html).
1. [Set up the starter workspace](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html#using-the-starter-workspace).
    - **Important**: Don't forget to `git clone --recursive` or `git submodule update --init --remote`, so that you obtain the standard query libraries.
1. Open the starter workspace: File > Open Workspace > Browse to `vscode-codeql-starter/vscode-codeql-starter.code-workspace`.
1. Download the [esbena_bootstrap-pre-27047_javascript CodeQL database](https://github.com/githubsatelliteworkshops/codeql/releases/download/v1.0/esbena_bootstrap-pre-27047_javascript.zip).
1. Unzip the database.
1. Import the unzipped database into Visual Studio Code:
    - Click the **CodeQL** icon in the left sidebar.
    - Place your mouse over **Databases**, and click the + sign that appears on the right.
    - Choose the unzipped database directory on your filesystem.
1. Create a new file, name it `UnsafeDollarCall.ql`, save it under `codeql-custom-queries-javascript`.

## Documentation links <a id="documentationlinks"></a>
If you get stuck, try searching our documentation and blog posts for help and ideas. Below are a few links to help you get started:
- [Learning CodeQL](https://help.semmle.com/QL/learn-ql)
- [Learning CodeQL for JavaScript](https://help.semmle.com/QL/learn-ql/javascript/ql-for-javascript.html)
- [Using the CodeQL extension for VS Code](https://help.semmle.com/codeql/codeql-for-vscode)

## Workshop <a id="workshop"></a>

The workshop is split into several steps. You can write one query per step, or work with a single query that you refine at each step.

Each step has a **Hint** that describe useful classes and predicates in the CodeQL standard libraries for JavaScript and keywords in CodeQL. You can explore these in your IDE using the autocomplete suggestions (`Ctrl+Space`) and jump-to-definition command (`F12`).

Each step has a **Solution** that indicates one possible answer. Note that all queries will need to begin with `import javascript`, but for simplicity this may be omitted below.

### Finding calls to the jQuery `$` function <a id="section1"></a>

1. Find all function call expressions, such as `alert("hello world")` and `speaker.sayHello("world")`.
    <details>
    <summary>Hint</summary>

    A function call is called a `CallExpr` in the CodeQL JavaScript library.
    </details>
     <details>
    <summary>Solution</summary>
    
    ```ql
    from CallExpr dollarCall
    select dollarCall
    ```
    </details>

1. Identify the expression that is used as the first argument for each call, , such as `alert(<first-argument>)` and `speaker.sayHello(<first-argument>)`.

    <details>
    <summary>Hint</summary>

    - Add another variable to your `from` clause. This can be named `dollarArg` and have type `Expr`.
    - Add a `where` clause.
    - `CallExpr` has a predicate `getArgument(int)` to find the argument at a 0-based index.
    </details>
    <details>
    <summary>Solution</summary>
    
    ```ql
    from CallExpr dollarCall, Expr dollarArg
    where dollarArg = dollarCall.getArgument(0)
    select dollarArg
    ```
    </details>

1. Filter your results to only those calls to a function named `$`, such as `$("hello world")` and `speaker.$("world")`.

    <details>
    <summary>Hint</summary>

    - `CallExpr` has a predicate `getCalleeName()` to find the name of the function being called.
    - Use the `and` keyword to add conditions to your query.
    - Use the `=` operator to assert that two values are equal.
    </details><details>
    <summary>Solution</summary>
    
    ```ql
    from CallExpr dollarCall, Expr dollarArg
    where
      dollarArg = dollarCall.getArgument(0) and
      dollarCall.getCalleeName() = "$"
    select dollarArg
    ```
    </details>

1. So far we have looked for the function name `$`. Are there other ways of calling the jQuery `$` function? Perhaps the CodeQL library can handle these for us?

    The CodeQL standard library for JavaScript has a built-in predicate `jquery()` to describe references to `$`. Expand the hint for details, and modify your query to use it.
    <details>
    <summary>Hint</summary>

    - Calling the predicate `jquery()` returns all values that refer to the `$` function.
    - To find all calls to this function, use the predicate `getACall()`.
    - Notice that when you call `jquery()`, `getACall()`, and `getAnArgument()` in succession, you get return values of type `DataFlow::Node`, not `Expr`. These are **data flow nodes**. They describe a part of the source program that may have a value, and let us do more complex reasoning about this value. We'll learn more about these in the next section.
    - You can change your `dollarArg` variable to have type `DataFlow::Node`, or convert the data flow node back into an `Expr` using the predicate `asExpr()`.
    </details><details>
    <summary>Solution</summary>
    
    ```ql
    from Expr dollarArg
    where
      dollarArg = jquery().getACall().getArgument(0).asExpr()
    select dollarArg
    ```

    OR

    ```ql
    from DataFlow::Node dollarArg
    where
      dollarArg = jquery().getACall().getArgument(0)
    select dollarArg
    ```
    
    </details>

### Finding jQuery plugin options <a id="section2"></a>
jQuery plugins are usually defined by assigning a value to a property of the `$.fn` object:

  ```javascript
  $.fn.copyText = function() { ... } // this function is a jQuery plugin
  ```

In this step, we will find such plugins, and their options.

Consider creating a new query for these next few steps, or commenting out your earlier solutions and using the same file. We will use the earlier solutions again in the next section.

1. You have already seen how to find references to the jQuery `$` function. Now find all places in the code that read the property `$.fn`.
    <details>
    <summary>Hint</summary>
    - Declare a new variable of type `DataFlow::Node` to hold the results.
    - Notice that `jQuery()` returns a value of type `DataFlow::SourceNode`. Source nodes are places in the program that introduce a new value, from which the flow of data may be tracked.
    - `DataFlow::SourceNode` has a predicate named `getAPropertyRead(string)`, which finds all reads of a particular property on the same object. The string argument is the name of the property.
    </details>
    <details>
    <summary>Solution</summary>
    
    ```ql
    from DataFlow::Node n
    where n = jquery().getAPropertyRead("fn")
    select n
    ```
    </details>

1. Find the functions that are assigned to a property of `$.fn`. These are jQuery plugins.

    Remember the previous example:
    ```javascript
    $.fn.copyText = function() { ... } // this function is a jQuery plugin
    ```

    There might be some variation in how this code is written. For example, we might see intermediate assignments to local variables:
     
    ```javascript
    let fn = $.fn
    let f = function() { ... } // this function is a jQuery plugin
    fn.copyText = f
    ```

    The use of intermediate variables and nested expressions are typical source code examples that require use of **local data flow analysis** to detect.

    Data flow analysis helps us answer questions like: does this expression ever hold a value that originates from a particular other place in the program?

    We have already encountered **data flow nodes**, described by the `DataFlow::Node` CodeQL class. They are places in the program that have a value. They are returned by useful predicates like `jquery()` in the library. 

    These nodes are separate and distinct from the AST (Abstract Syntax Tree, which represents the basic structure of the program) nodes, to allow for flexibility in how data flow is modeled.

    We can visualize the data flow analysis problem as one of finding paths through a directed graph, where the nodes of the graph are data flow nodes, and the edges represent the flow of data between those elements. If a path exists, then the data flows between those two nodes.
    
    The CodeQL JavaScript data flow library is very expressive.
    It has several classes that describe different places in the program that can have a value. We have seen `SourceNode`s; there are many other forms such as `ValueNode`s, `FunctionNode`s, `ParameterNode`s, and `CallNode`s. You can find our more in the [documentation](https://help.semmle.com/QL/learn-ql/javascript/dataflow.html).
    
    When we are looking for the flow of information to or from these nodes within a single function or scope, this is called **local data flow analysis**. The CodeQL library has several predicates available on different types of data flow node that reason about local data flow.

    You have already seen one such predicate: `SourceNode.getAPropertyRead()`. To complete this step of the workshop, look at the hint for another useful predicate.

    <details>
    <summary>Hint</summary>

    - `DataFlow::SourceNode` has a predicate named `getAPropertySource()`, which finds a source node whose value is stored in a property of this node.
    - In the previous step, we used `getAPropertyRead(string)` to identify the source node `$.fn`. Now try to find a value stored in a property of this source node `$.fn`.
    
    </details>
    <details>
    <summary>Solution</summary>
    
    ```ql
    from DataFlow::Node plugin
    where plugin = jquery().getAPropertyRead("fn").getAPropertySource()
    select plugin
    ```
    </details>

1. Find the last parameter of the jQuery plugin functions that you identified in the previous step. These parameters are the plugin options.

    <details>
    <summary>Hint</summary>

    - Modify your `from` clause so that the variable that describes that jQuery plugin is of type `DataFlow::FunctionNode`. As the name suggests, this is a data flow node that refers to a function definition.
    - `DataFlow::FunctionNode` has a predicate named `getLastParameter()`.
    - If you want to add a new variable to describe the parameter, it can be of type `DataFlow::ParameterNode`.
    
    </details>
    <details>
    <summary>Solution</summary>
    
    ```ql
    from DataFlow::FunctionNode plugin, DataFlow::ParameterNode optionsParam
    where
      plugin = jquery().getAPropertyRead("fn").getAPropertySource() and
      optionsParam = plugin.getLastParameter()
    select plugin, optionsParam
    ```
    </details>

### Putting it all together <a id="section3"></a>

We have now identified (a) places in the program which receive jQuery plugin options (which may be untrusted data) and (b) places in the program which are passed to the jQuery `$` function and may be interpreted as HTML. We now want to tie these two together to ask: does the untrusted data from a jQuery plugin option ever _flow_ to the potentially unsafe `$` call?

This is also a data flow problem. However, it is larger in scope that the problems we have tackled so far, because the plugin options and the `$` call may be in different functions. We call this a **global data flow** problem.

In this section we will create a  _path problem_ query capable of looking for global data flow, by populating this template:

```ql
/**
 * @name Cross-site scripting vulnerable plugin
 * @kind path-problem
 * @id js/xss-unsafe-plugin
 */
import javascript
import DataFlow::PathGraph

class Config extends TaintTracking::Configuration {
  Config() { this = "Config" }
  override predicate isSource(DataFlow::Node source) {
    exists(/** TODO fill me in from Section 2 **/ |
      source = /** TODO fill me in from Section 2 **/
    )
  }
  override predicate isSink(DataFlow::Node sink) {
    sink = /** TODO fill me in from Section 1 **/
  }
}

from Config config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink, source, sink, "Potential XSS vulnerability in plugin."
```

 1. Complete the `isSource` predicate using the query you wrote for [Section 2](#section2).

    <details>
    <summary>Hint</summary>

    - You can translate from a query clause to a predicate by:
       - Converting the variable declarations in the `from` part to the variable declarations of an `exists`
       - Placing the `where` clause conditions (if any) in the body of the exists
       - Adding a condition which equates the `select` to one of the parameters of the predicate.
    - Remember that the source of untrusted data is the jQuery plugin options parameter.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
      override predicate isSource(DataFlow::Node source) {
        exists(DataFlow::FunctionNode plugin |
          plugin = jquery().getAPropertyRead("fn").getAPropertySource() and
          source = plugin.getLastParameter()
        )
      }
    ```
    </details>

 1. Complete the `isSink` predicate by using the query you wrote for [Section 1](#section1).
    <details>
    <summary>Hint</summary>

    - Complete the same process as above.
    - We already found a `DataFlow::Node` in Section 1 as the result of calling `jquery()` and predicates on it.
    - Remember that the first argument of a call to `$` is a sink for XSS vulnerabilities.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
      override predicate isSink(DataFlow::Node sink) {
        sink = jquery().getACall().getArgument(0)
      }
    ```
    </details>

1. You can now run the completed query. This should find 5 results on the unpatched Bootstrap codebase.

    <details>
    <summary>Completed query</summary>

      ```ql
      /**
      * @name Cross-site scripting vulnerable plugin
      * @kind path-problem
      * @id js/xss-unsafe-plugin
      */

      import javascript
      import DataFlow::PathGraph

      class Configuration extends TaintTracking::Configuration {
        Configuration() { this = "XssUnsafeJQueryPlugin" }

        override predicate isSource(DataFlow::Node source) {
          exists(DataFlow::FunctionNode plugin |
            plugin = jquery().getAPropertyRead("fn").getAPropertySource() and
            source = plugin.getLastParameter()
          )
        }

        override predicate isSink(DataFlow::Node sink) {
          sink = jquery().getACall().getArgument(0)
        }
      }

      from Configuration cfg, DataFlow::PathNode source, DataFlow::PathNode sink
      where cfg.hasFlowPath(source, sink)
      select sink, source, sink, "Potential XSS vulnerability in plugin."
      ```
    </details>

## What's next?
- Read the [tutorial on analyzing data flow in JavaScript and TypeScript](https://help.semmle.com/QL/learn-ql/javascript/dataflow.html).
- Try out the latest CodeQL Capture-the-Flag challenge on the [GitHub Security Lab website](https://securitylab.github.com/ctf) for a chance to win a prize! Or try one of the older Capture-the-Flag challenges to improve your CodeQL skills.
- Try out a CodeQL course on [GitHub Learning Lab](https://lab.github.com/githubtraining/codeql-u-boot-challenge-(cc++)).

## Acknowledgements

This is a reduced version of a Capture-the-Flag challenge devised by @esbena, available at https://securitylab.github.com/ctf/jquery. Try out the full version! Thanks to our moderators for valuable feedback on the workshop.
