---
highlight: a11y-dark
---
最进在做工程模版自动生成的任务时，实用到了Java解析工具**javaparser**特此记录一下  
这是一个开源工具，地址 https://github.com/javaparser/javaparser  
具体语法我就不讲了，也没有太好的文档，就摸索着能写一些简单的代码

### 1. 简单使用：完全靠代码生成一个类文件

```
CompilationUnit cu = new CompilationUnit();
cu.setPackageDeclaration("example.model");
cu.addImport("java.util.List");

ClassOrInterfaceDeclaration book = cu.addClass("Book");
book.addField("String", "title");
book.addField("Person", "author");
book.addConstructor(Modifier.Keyword.PUBLIC)
        .addParameter("String", "title")
        .addParameter("Person", "author")
        .setBody(new BlockStmt()
                .addStatement(new ExpressionStmt(new AssignExpr(
                        new FieldAccessExpr(new ThisExpr(), "title"),
                        new NameExpr("title"),
                        AssignExpr.Operator.ASSIGN)))
                .addStatement(new ExpressionStmt(new AssignExpr(
                        new FieldAccessExpr(new ThisExpr(), "author"),
                        new NameExpr("author"),
                        AssignExpr.Operator.ASSIGN))));

book.addMethod("getTitle", Modifier.Keyword.PUBLIC).setBody(
        new BlockStmt().addStatement(new ReturnStmt(new NameExpr("title"))));

book.addMethod("getAuthor", Modifier.Keyword.PUBLIC).setBody(
        new BlockStmt().addStatement(new ReturnStmt(new NameExpr("author"))));

System.out.println(cu.toString());

```

生成的文件的样子

```
package example.model;

import java.util.List;

public class Book {

    String title;

    Person author;

    public Book(String title, Person author) {
        this.title = title;
        this.author = author;
    }

    public void getTitle() {
        return title;
    }

    public void getAuthor() {
        return author;
    }
}
```

### 2. 简单使用：靠代码和一部分自己拼接的字符串生成一个类文件。下面代码随意组合，应该能满足大部分场景了

```
CompilationUnit cu = new CompilationUnit();
cu.setPackageDeclaration("com.xwy.example");
cu.addImport("java.util.List");

ClassOrInterfaceDeclaration registerFile = cu.addClass("Car", Modifier.Keyword.FINAL);
registerFile.addExtendedType("Object");

ReturnStmt returnStmt = new ReturnStmt();
NameExpr returnNameExpr = new NameExpr();
returnNameExpr.setName("i");
returnStmt.setExpression(returnNameExpr);


registerFile.addMethod("toString", Modifier.Keyword.PUBLIC)
        .addAnnotation("Override")
        .setType("String")
        .setBody(new BlockStmt().
                addStatement(new ReturnStmt().
                        setExpression(new NameExpr()
                                .setName("super.toString()"))));

registerFile.addMethod("init", Modifier.Keyword.PUBLIC)
        .addParameter("int", "i")
        .setType("int")
        .setBody(new BlockStmt().addStatement(returnStmt));
System.out.println(cu.toString());
```
生成的文件的样子

```
package com.xwy.example;

import java.util.List;

final class Car extends Object {

    @Override()
    public String toString() {
        return super.toString();
    }

    public int init(int i) {
        return i;
    }
}
```
### 3. 读文件和添加代码

```
String path = "/Users/xuweiyu/IdeaProjects/AST/src/main/java/LeanParserConfiguration.java";
JavaParser javaParser = new JavaParser();
ParseResult<CompilationUnit> result = javaParser.parse(new File(path));
result.getResult().get().getTypes().forEach(type -> {
    System.out.println(type.getName());
    //获取类名
    type.getMethods().forEach(method -> {
        System.out.println(method.getName());
        //获取方法名
        Iterator<Statement> iterator = method.getBody().get().getStatements().iterator();
        while (iterator.hasNext()) {
            Statement statement = iterator.next();
            if (statement.toString().contains("789"))
                iterator.remove();
            else
                System.out.println("statement:" + statement.toString());
        }
        //此处在读文件的过程中添加了一行代码
        Expression whileCounterExpression = StaticJavaParser
                .parseVariableDeclarationExpr(" int i " + " = 0");
        whileCounterExpression.setLineComment("这是个注释，这里是新增的代码");
        method.getBody().get().addStatement(whileCounterExpression);
    });
});
System.out.println(result.getResult().get().toString());
```

读出来的样子

```
import com.github.javaparser.ParserConfiguration;

public class LeanParserConfiguration extends ParserConfiguration {

    public LeanParserConfiguration() {
        setLanguageLevel(ParserConfiguration.LanguageLevel.RAW);
        setStoreTokens(false);
        setAttributeComments(false);
        setLexicalPreservationEnabled(false);
    }

    public void f1() {
        System.out.printf("123");
        System.out.printf("456");
        // 这是个注释，这里是新增的代码
        int i = 0;
    }
}
```

源文件

```
import com.github.javaparser.ParserConfiguration;
import java.util.List;

public class LeanParserConfiguration extends ParserConfiguration {

    public LeanParserConfiguration() {
        setLanguageLevel(ParserConfiguration.LanguageLevel.RAW);
        setStoreTokens(false);
        setAttributeComments(false);
        setLexicalPreservationEnabled(false);
    }

    public void f1() {
        System.out.printf("123");
        System.out.printf("456");
    }
}
```

基本操作应该都包含了

参考文档：https://zhuanlan.zhihu.com/p/57545063
