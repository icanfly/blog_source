---
title: groovy-expr-usage
tags:
  - groovy
  - 正则表达式
categories:
  - 原创文章
originContent: >-
  在开发单据规则计算引擎的时候引入了groovy脚本计算引擎，其中有一个规则函数：正则函数需要使用正则表达式计算，顺便找了一下groovy里的正则表达式：


  groovy中对于正则表达式的书写进行了简化，同时它仍然是引用的java核心的正则表达式引擎，并没有自己实现一套正则引擎，更多的是从语法糖的形式上进行优化，让人使用起来格外的舒服。


  - 查找（find）操作符：=~  返回Matcher类型

  - 匹配（match）操作符：==~  返回boolean类型

  - 模式(pattern)操作符：~String  返回Pattern类型


  ```groovy

  class ExprCheck implements FunctionInvoke {

      def EXPR_PARAM = "expr";

      FunctionResult invoke(FunctionContext ctx) {
          def currentVal = ctx.currentVal;
          def exprStr = ctx.systemParams.get(EXPR_PARAM);
          def expr = ~exprStr;
          return new FunctionResult(currentVal ==~ expr);
      }

  }


  ```


  测试：

  ```groovy

  class ExprCheckTest {

      @Test
      public void test1() {
          FunctionContext ctx = new FunctionContext();
          ctx.currentVal = "hello"
          ctx.systemParams = ["expr":"hello"]

          ExprCheck check = new ExprCheck();
          FunctionResult result = check.invoke(ctx);
          assertTrue(result.valid);
      }

      @Test
      public void test2() {
          FunctionContext ctx = new FunctionContext();
          ctx.currentVal = "hello1"
          ctx.systemParams = ["expr":"hellod+"]

          ExprCheck check = new ExprCheck();
          FunctionResult result = check.invoke(ctx);
          assertTrue(!result.valid);
      }

      @Test
      public void test3() {
          FunctionContext ctx = new FunctionContext();
          ctx.currentVal = "hello1"
          ctx.systemParams = ["expr":"hello\\d+"]

          ExprCheck check = new ExprCheck();
          FunctionResult result = check.invoke(ctx);
          assertTrue(result.valid);
      }

  }


  ```
toc: false
date: 2019-01-08 19:23:32
author:
thumbnail:
blogexcerpt:
---

在开发单据规则计算引擎的时候引入了groovy脚本计算引擎，其中有一个规则函数：正则函数需要使用正则表达式计算，顺便找了一下groovy里的正则表达式：

groovy中对于正则表达式的书写进行了简化，同时它仍然是引用的java核心的正则表达式引擎，并没有自己实现一套正则引擎，更多的是从语法糖的形式上进行优化，让人使用起来格外的舒服。

- 查找（find）操作符：=~  返回Matcher类型
- 匹配（match）操作符：==~  返回boolean类型
- 模式(pattern)操作符：~String  返回Pattern类型

<!--more-->

```groovy
class ExprCheck implements FunctionInvoke {

    def EXPR_PARAM = "expr";

    FunctionResult invoke(FunctionContext ctx) {
        def currentVal = ctx.currentVal;
        def exprStr = ctx.systemParams.get(EXPR_PARAM);
        def expr = ~exprStr;
        return new FunctionResult(currentVal ==~ expr);
    }

}

```

测试：
```groovy
class ExprCheckTest {

    @Test
    public void test1() {
        FunctionContext ctx = new FunctionContext();
        ctx.currentVal = "hello"
        ctx.systemParams = ["expr":"hello"]

        ExprCheck check = new ExprCheck();
        FunctionResult result = check.invoke(ctx);
        assertTrue(result.valid);
    }

    @Test
    public void test2() {
        FunctionContext ctx = new FunctionContext();
        ctx.currentVal = "hello1"
        ctx.systemParams = ["expr":"hellod+"]

        ExprCheck check = new ExprCheck();
        FunctionResult result = check.invoke(ctx);
        assertTrue(!result.valid);
    }

    @Test
    public void test3() {
        FunctionContext ctx = new FunctionContext();
        ctx.currentVal = "hello1"
        ctx.systemParams = ["expr":"hello\\d+"]

        ExprCheck check = new ExprCheck();
        FunctionResult result = check.invoke(ctx);
        assertTrue(result.valid);
    }

}

```