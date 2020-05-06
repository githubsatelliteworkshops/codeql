# CodeQL workshop for Java: Unsafe deserialization in Apache Struts

 - Analyzed language: Java
 - Difficulty level: 200

## Overview

 - [Problem statement](#problemstatement)
 - [Setup instructions](#setupinstructions)
 - [Documentation links](#documentationlinks)
 - [Workshop](#workshop)
   - [Section 1: Finding XML deserialization](#section1)
   - [Section 2: Find the implementations of the `toObject` method from ContentTypeHandler](#section2)
   - [Section 3: Unsafe XML deserialization](#section3)

## Problem statement <a id="problemstatement"></a>

_Serialization_ is the process of converting in memory objects to text or binary output formats, usually for the purpose of sharing or saving program state. This serialized data can then be loaded back into memory at a future point through the process of _deserialization_.

In languages such as Java, Python and Ruby, deserialization provides the ability to restore not only primitive data, but also complex types such as library and user defined classes. This provides great power and flexibility, but introduces a signficant attack vector if the deserialization happens on untrusted user data without restriction.

[Apache Struts](https://struts.apache.org/) is a popular open-source MVC framework for creating web applications in Java. In 2017, a researcher from the predecessor of the [GitHub Security Lab](https://securitylab.github.com/) found [CVE-2017-9805](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-9805), an XML deserialization vulnerability in Apache Struts that would allow remote code execution.

The problem occurred because included as part of the Apache Struts framework is the ability to accept requests in multiple different formats, or _content types_. It provides a pluggable system for supporting these content types through the [`ContentTypeHandler`](https://struts.apache.org/maven/struts2-plugins/struts2-rest-plugin/apidocs/org/apache/struts2/rest/handler/ContentTypeHandler.html) interface, which provides the following interface method:
```java
    /**
     * Populates an object using data from the input stream
     * @param in The input stream, usually the body of the request
     * @param target The target, usually the action class
     * @throws IOException If unable to write to the output stream
     */
    void toObject(Reader in, Object target) throws IOException;
```
New content type handlers are defined by implementing the interface and defining a `toObject` method which takes data in the specified content type (in the form of a `Reader`) and uses it to populate the Java object `target`, often via a deserialization routine. However, the `in` parameter is typically populated from the body of a request without sanitization or safety checks. This means it should be treated as "untrusted" user data, and only deserialized under certain safe conditions.

In this workshop, we will write a query to find CVE-2017-9805 in a database built from the known vulnerable version of Apache Struts.

## Setup instructions for Visual Studio Code <a id="setupinstructions"></a>

To take part in the workshop you will need to follow these steps to get the CodeQL development environment setup:

1. Install the Visual Studio Code IDE.
2. Download and install the [CodeQL extension for Visual Studio Code](https://help.semmle.com/codeql/codeql-for-vscode.html). Full setup instructions are [here](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html).
3. [Set up the starter workspace](https://help.semmle.com/codeql/codeql-for-vscode/procedures/setting-up.html#using-the-starter-workspace).
    -   ****Important****: Don't forget to `git clone --recursive` or `git submodule update --init --remote`, so that you obtain the standard query libraries.
4. Open the starter workspace: File > Open Workspace > Browse to `vscode-codeql-starter/vscode-codeql-starter.code-workspace`.
5. Download and unzip the [apache-struts-91ae344-CVE-2017-9805 database](https://downloads.lgtm.com/snapshots/java/apache/struts/apache-struts-91ae344-CVE-2017-9805.zip).
6. Choose this database in CodeQL (using `Ctrl + Shift + P` to open the command palette, then selecting "CodeQL: Choose Database").
7. Create a new file in the `codeql-custom-queries-java` directory called `UnsafeDeserialization.ql`.

## Documentation links <a id="documentationlinks"></a>
If you get stuck, try searching our documentation and blog posts for help and ideas. Below are a few links to help you get started:
- [Learning CodeQL](https://help.semmle.com/QL/learn-ql)
- [Learning CodeQL for Java](https://help.semmle.com/QL/learn-ql/cpp/ql-for-java.html)
- [Using the CodeQL extension for VS Code](https://help.semmle.com/codeql/codeql-for-vscode.html)

## Workshop <a id="workshop"></a>

The workshop is split into several steps. You can write one query per step, or work with a single query that you refine at each step. Each step has a **hint** that describes useful classes and predicates in the CodeQL standard libraries for Java. You can explore these in your IDE using the autocomplete suggestions (`Ctrl + Space`) and the jump-to-definition command (`F12`).

### Section 1: Finding XML deserialization <a id="section1"></a>

[XStream](https://x-stream.github.io/index.html) is a Java framework for serializing Java objects to XML used by Apache Struts. It provides a method `XStream.fromXML` for deserializing XML to a Java object. By default, the input is not validated in any way, and is vulnerable to remote code execution exploits. In this section, we will identify calls to `fromXML` in the codebase.

 1. Find all method calls in the program.
    <details>
    <summary>Hint</summary>

    - A method call is represented by the `MethodAccess` type in the CodeQL Java library.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    import java

    from MethodAccess call
    select call
    ```
    </details>

 1. Update your query to report the method being called by each method call.
    <details>
    <summary>Hints</summary>

    - Add a CodeQL variable called `method` with type `Method`.
    - `MethodAccess` has a predicate called `getMethod()` for returning the method.
    - Add a `where` clause.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    import java

    from MethodAccess call, Method method
    where call.getMethod() = method
    select call, method
    ```
    </details>

 1. Find all calls in the program to methods called `fromXML`.<a id="question1"></a>

    <details>
    <summary>Hint</summary>

    - `Method.getName()` returns a string representing the name of the method.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    import java

    from MethodAccess fromXML, Method method
    where
      fromXML.getMethod() = method and
      method.getName() = "fromXML"
    select fromXML
    ```
    However, as we now want to report only the call itself, we can inline the temporary `method` variable like so:
    ```ql
    import java

    from MethodAccess fromXML
    where fromXML.getMethod().getName() = "fromXML"
    select fromXML
    ```
    </details>

 1. The `XStream.fromXML` method deserializes the first argument (i.e. the argument at index `0`). Update your query to report the deserialized argument.

    <details>
    <summary>Hint</summary>

    - `MethodCall.getArgument(int i)` returns the argument at the i-th index.
    - The arguments are _expressions_ in the program, represented by the CodeQL class `Expr`. Introduce a new variable to hold the argument expression.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    import java

    from MethodAccess fromXML, Expr arg
    where
      fromXML.getMethod().getName() = "fromXML" and
      arg = fromXML.getArgument(0)
    select fromXML, arg
    ```
    </details>

 1. Recall that _predicates_ allow you to encapsulate logical conditions in a reusable format. Convert your previous query to a predicate which identifies the set of expressions in the program which are deserialized directly by `fromXML`. You can use the following template:
    ```ql
    predicate isXMLDeserialized(Expr arg) {
      exists(MethodAccess fromXML |
        // TODO fill me in
      )
    }
    ```
    [`exists`](https://help.semmle.com/QL/ql-handbook/formulas.html#exists) is a mechanism for introducing temporary variables with a restricted scope. You can think of them as their own `from`-`where`-`select`. In this case, we use it to introduce the `fromXML` temporary variable, with type `MethodAccess`.

    <details>
    <summary>Hint</summary>

     - Copy the `where` clause of the previous query.
    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    import java

    predicate isXMLDeserialized(Expr arg) {
      exists(MethodAccess fromXML |
        fromXML.getMethod().getName() = "fromXML" and
        arg = fromXML.getArgument(0)
      )
    }

    from Expr arg
    where isXMLDeserialized(arg)
    select arg
    ```

### Section 2: Find the implementations of the toObject method from ContentTypeHandler <a id="section2"></a>

Like predicates, _classes_ in CodeQL can be used to encapsulate reusable portions of logic. Classes represent single sets of values, and they can also include operations (known as _member predicates_) specific to that set of values. You have already seen numerous instances of CodeQL classes (`MethodAccess`, `Method` etc.) and associated member predicates (`MethodAccess.getMethod()`, `Method.getName()`, etc.).

 1. Create a CodeQL class called `ContentTypeHandler` to find the interface `org.apache.struts2.rest.handler.ContentTypeHandler`. You can use this template:
    ```ql
    class ContentTypeHandler extends RefType {
      ContentTypeHandler() {
          // TODO Fill me in
      }
    }
    ```

    <details>
    <summary>Hint</summary>

    - Use `RefType.hasQualifiedName(string packageName, string className)` to identify classes with the given package name and class name. For example:
        ```ql
        from RefType r
        where r.hasQualifiedName("java.lang", "String")
        select r
        ```
    - Within the characteristic predicate you can use the magic variable `this` to refer to the RefType

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    import java

    /** The interface `org.apache.struts2.rest.handler.ContentTypeHandler`. */
    class ContentTypeHandler extends RefType {
      ContentTypeHandler() {
        this.hasQualifiedName("org.apache.struts2.rest.handler", "ContentTypeHandler")
      }
    }
    ```
    </details>

 2. Create a CodeQL class called `ContentTypeHandlerToObject` for identfying `Method`s called `toObject` on classes whose direct super-types include `ContentTypeHandler`.

    <details>
    <summary>Hint</summary>

    - Use `Method.getName()` to identify the name of the method.
    - To identify whether the method is declared on a class whose direct super-type includes `ContentTypeHandler`, you will need to:
      - Identify the declaring type of the method using `Method.getDeclaringType()`.
      - Identify the super-types of that type using `RefType.getASuperType()`
      - Use `instanceof` to assert that one of the super-types is a `ContentTypeHandler`

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    /** A `toObject` method on a subtype of `org.apache.struts2.rest.handler.ContentTypeHandler`. */
    class ContentTypeHandlerToObject extends Method {
      ContentTypeHandlerToObject() {
        this.getDeclaringType().getASupertype() instanceof ContentTypeHandler and
        this.hasName("toObject")
      }
    }
    ```
    </details>

 3. `toObject` methods should consider the first parameter as untrusted user input. Write a query to find the first (i.e. index 0) parameter for `toObject` methods.
    <details>
    <summary>Hint</summary>

    - Use `Method.getParameter(int index)` to get the i-th index parameter.
    - Create a query with a single CodeQL variable of type `ContentTypeHandlerToObject`.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
    from ContentTypeHandlerToObject toObjectMethod
    select toObjectMethod.getParameter(0)
    ```
    </details>

### Section 3: Unsafe XML deserialization <a id="section3"></a>

We have now identified (a) places in the program which receive untrusted data and (b) places in the program which potentially perform unsafe XML deserialization. We now want to tie these two together to ask: does the untrusted data ever _flow_ to the potentially unsafe XML deserialization call?

In program analysis we call this a _data flow_ problem. Data flow helps us answer questions like: does this expression ever hold a value that originates from a particular other place in the program?

We can visualize the data flow problem as one of finding paths through a directed graph, where the nodes of the graph are elements in program, and the edges represent the flow of data between those elements. If a path exists, then the data flows between those two nodes.

Consider this example Java method:

```c
int func(int tainted) {
   int x = tainted;
   if (someCondition) {
     int y = x;
     callFoo(y);
   } else {
     return x;
   }
   return -1;
}
```
The data flow graph for this method will look something like this:

<img src="https://help.semmle.com/QL/ql-training/_images/graphviz-2ad90ce0f4b6f3f315f2caf0dd8753fbba789a14.png" alt="drawing" width="260"/>

This graph represents the flow of data from the tainted parameter. The nodes of graph represent program elements that have a value, such as function parameters and expressions. The edges of this graph represent flow through these nodes.

CodeQL for Java provides data flow analysis as part of the standard library. You can import it using `semmle.code.java.dataflow.DataFlow`. The library models nodes using the `DataFlow::Node` CodeQL class. These nodes are separate and distinct from the AST (Abstract Syntax Tree, which represents the basic structure of the program) nodes, to allow for flexibility in how data flow is modeled.

There are a small number of data flow node types – expression nodes and parameter nodes are most common.

In this section we will create a data flow query by populating this template:

```ql
/**
 * @name Unsafe XML deserialization
 * @kind problem
 * @id java/unsafe-deserialization
 */
import java
import semmle.code.java.dataflow.DataFlow

// TODO add previous class and predicate definitions here

class StrutsUnsafeDeserializationConfig extends DataFlow::Configuration {
  StrutsUnsafeDeserializationConfig() { this = "StrutsUnsafeDeserializationConfig" }
  override predicate isSource(DataFlow::Node source) {
    exists(/** TODO fill me in **/ |
      source.asParameter() = /** TODO fill me in **/
    )
  }
  override predicate isSink(DataFlow::Node sink) {
    exists(/** TODO fill me in **/ |
      /** TODO fill me in **/
      sink.asExpr() = /** TODO fill me in **/
    )
  }
}

from StrutsUnsafeDeserializationConfig config, DataFlow::Node source, DataFlow::Node sink
where config.hasFlow(source, sink)
select sink, "Unsafe XML deserialization"
```

 1. Complete the `isSource` predicate using the query you wrote for [Section 2](#section2).

    <details>
    <summary>Hint</summary>

    - You can translate from a query clause to a predicate by:
       - Converting the variable declarations in the `from` part to the variable declarations of an `exists`
       - Placing the `where` clause conditions (if any) in the body of the exists
       - Adding a condition which equates the `select` to one of the parameters of the predicate.
    - Remember to include the `ContentTypeHandlerToObject` class you defined earlier.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
      override predicate isSource(Node source) {
        exists(ContentTypeHandlerToObject toObjectMethod |
          source.asParameter() = toObjectMethod.getParameter(0)
        )
      }
    ```
    </details>

 1. Complete the `isSink` predicate by using the final query you wrote for [Section 1](#section1). Remember to use the `isXMLDeserialized` predicate!
    <details>
    <summary>Hint</summary>

    - Complete the same process as above.

    </details>
    <details>
    <summary>Solution</summary>

    ```ql
      override predicate isSink(Node sink) {
        exists(Expr arg |
          isXMLDeserialized(arg) and
          sink.asExpr() = arg
        )
      }
    ```
    </details>

You can now run the completed query. You should find exactly one result, which is the CVE reported by our security researchers in 2017!

For this result, it is easy to verify that it is correct, because both the source and sink are in the same method. However, for many data flow problems this is not the case.

We can update the query so that it not only reports the sink, but it also reports the source and the path to that source. We can do this by making these changes:
The answer to this is to convert the query to a _path problem_ query. There are five parts we will need to change:
 - Convert the `@kind` from `problem` to `path-problem`. This tells the CodeQL toolchain to interpret the results of this query as path results.
 - Add a new import `DataFlow::PathGraph`, which will report the path data alongside the query results.
 - Change `source` and `sink` variables from `DataFlow::Node` to `DataFlow::PathNode`, to ensure that the nodes retain path information.
 - Use `hasFlowPath` instead of `hasFlow`.
 - Change the select to report the `source` and `sink` as the second and third columns. The toolchain combines this data with the path information from `PathGraph` to build the paths.

 3. Convert your previous query to a path-problem query.
    <details>
    <summary>Solution</summary>

    ```ql
    /**
    * @name Unsafe XML deserialization
    * @kind path-problem
    * @id java/unsafe-deserialization
    */
    import java
    import semmle.code.java.dataflow.DataFlow
    import DataFlow::PathGraph

    /** The interface `org.apache.struts2.rest.handler.ContentTypeHandler`. */
    class ContentTypeHandler extends RefType {
      ContentTypeHandler() {
        this.hasQualifiedName("org.apache.struts2.rest.handler", "ContentTypeHandler")
      }
    }

    /** A `toObject` method on a subtype of `org.apache.struts2.rest.handler.ContentTypeHandler`. */
    class ContentTypeHandlerToObject extends Method {
      ContentTypeHandlerToObject() {
        this.getDeclaringType().getASupertype() instanceof ContentTypeHandler and
        this.hasName("toObject")
      }
    }

    class StrutsUnsafeDeserializationConfig extends DataFlow::Configuration {
      StrutsUnsafeDeserializationConfig() { this = "StrutsUnsafeDeserializationConfig" }
      override predicate isSource(DataFlow::Node source) {
        exists(ContentTypeHandlerToObject toObjectMethod |
          source.asParameter() = toObjectMethod.getParameter(0)
        )
      }
      override predicate isSink(DataFlow::Node sink) {
        exists(Expr arg |
          isXMLDeserialized(arg) and
          sink.asExpr() = arg
        )
      }
    }

    from StrutsUnsafeDeserializationConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
    where config.hasFlowPath(source, sink)
    select sink, source, sink, "Unsafe XML deserialization"
    ```
    </details>

For more information on how the vulnerability was identified, you can read the [blog disclosing the original problem](https://securitylab.github.com/research/apache-struts-vulnerability-cve-2017-9805).

Although we have created a query from scratch to find this problem, it can also be found with one of our default security queries, [UnsafeDeserialization.ql](https://github.com/github/codeql/blob/master/java/ql/src/Security/CWE/CWE-502/UnsafeDeserialization.ql). You can see this on a [vulnerable copy of Apache Struts](https://github.com/m-y-mo/struts_9805) that has been [analyzed on LGTM.com](https://lgtm.com/projects/g/m-y-mo/struts_9805/snapshot/31a8d6be58033679a83402b022bb89dad6c6e330/files/plugins/rest/src/main/java/org/apache/struts2/rest/handler/XStreamHandler.java?sort=name&dir=ASC&mode=heatmap#x121788d71061ed86:1), our free open source analysis platform.

## Follow up material

 - [Tutorial: Analyzing data flow in Java](https://help.semmle.com/QL/learn-ql/java/dataflow.html)
 - [CodeQL training for Java](https://help.semmle.com/QL/learn-ql/ql-training.html#codeql-and-variant-analysis-for-java)
 - [GitHub Security Lab research blog](https://securitylab.github.com/research)