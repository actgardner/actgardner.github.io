---
layout: post
title:  "Comparing Spark Dataframe Columns"
date:   2016-09-30 00:00:00
categories: spark scala java testing dataframe
---

Spark DataFrames provide an API to operate on tabular data. The `Column` class represents a tree of operations to be applied to each input record: things like mathematical operations, comparisons, etc. We were writing some unit tests to ensure some of our code produces an appropriate `Column` for an input query, and we noticed something interesting. This test builds two identical, trivial `Column` trees - getting the value for some column in the record, then adding 1 to it:

```
Column val1 = df.col("col1").plus(1);
Column val2 = df.col("col1").plus(1);
Assert.assertEquals(val1, val2);
```

This test passes with flying colours: `val1` and `val2` represent identical operations, so they're equal. 

This equally trivial test builds the same trees, but it adds a renaming step where the columns in the output schema have user-defined names:

```
Column val1 = df.col("col1").plus(1).as("renamedCol");
Column val2 = df.col("col1").plus(1).as("renamedCol");
Assert.assertEquals(val1, val2);
```

This test fails. What?!? It's going to be pretty hard to write tests if two `Column` trees with identical operations aren't equal to each other, just because they've been renamed. 

Diving into the debugger, we see that:

- Spark `Columns` contain a Catalyst `Expression`
- The `Expression` is what's different between the two instances
- Specifically, the `Expression` is an `Alias`, which has a field `exprId`
- `exprId` appears to have a different, random value in each instance of `Alias`

Catalyst is Spark's optimizer, where the operations of the query are actually represented. Using my very limited Scala knowledge, I looked at the definition of a Catalyst `Alias`:

```
case class Alias(child: Expression, name: String)(
    val exprId: ExprId = NamedExpression.newExprId,
    val qualifiers: Seq[String] = Nil,
    val explicitMetadata: Option[Metadata] = None)
  extends UnaryExpression with NamedExpression
```

`Alias` has the trait `NamedExpression`. Looking at the definiton of `NamedExpression`, we find out culprit:

```
object NamedExpression {
  private val curId = new java.util.concurrent.atomic.AtomicLong()
  private[expressions] val jvmId = UUID.randomUUID()
  def newExprId: ExprId = ExprId(curId.getAndIncrement(), jvmId)
  def unapply(expr: NamedExpression): Option[(String, DataType)] = Some(expr.name, expr.dataType)
}
```

`exprId` is deliberately a JVM-wide unique value! That complicates things somewhat. For our use-case we want to be able to compare two `Alias` instances and ensure the name and the underlying `Expression` are the same, while ignoring the `exprId` field. The usual `equals` method doesn't cut it. 

Fortunately, Scala has a neat pattern - the Extractor pattern - which is basically a backwards constructor. Conventionally there's a method `unapply` which returns a tuple of the arguments used to construct the object. In the case of `Alias`, the return type for the extractor method is `Option<Tuple2<Expression, String>>` - which contains the exact fields we want to compare.

In conclusion, we can write our own, hacky Java equality method for `Column` objects:

```
  public static boolean columnsEqual(Column expected, Column actual) {
    if (expected.expr() instanceof Alias && actual.expr() instanceof Alias) {
      Alias expectedNExpr = (Alias)expected.expr();
      Alias actualNExpr = (Alias)actual.expr();
      Option<Tuple2<Expression, String>> expectedExpr = Alias$.MODULE$.unapply(expectedNExpr);
      Option<Tuple2<Expression, String>> actualExpr = Alias$.MODULE$.unapply(actualNExpr);
      return expectedExpr.get()._1.equals(actualExpr.get()._1) 
        && expectedExpr.get()._2.equals(actualExpr.get()._2);
    } else {
      // If the Expressions within the Columns aren't Aliases, 
      // they haven't been renamed and conventional equality is fine
      return expected.equals(actual);
    }
  }
```

One limitation of this approach is that only the roots of `expected` and `actual` can be renamed.
This method will correctly compare `df.col("col1").as("renamed")`, but Spark supports renaming any `Column` in the tree.
A `Column` like `df.col("col1").as("renamed").as("renamedAgain")` will not be compared correctly, because there are multiple renaming steps. 
