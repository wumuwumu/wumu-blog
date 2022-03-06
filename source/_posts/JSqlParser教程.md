---
title: JSqlParser教程
tags:
  - java
abbrlink: 965e3a7f
date: 2018-12-05 21:58:31
---

# 解析

## 获取表名

```java
//获取所有使用过的表
Statement statement = CCJSqlParserUtil.parse("SELECT * FROM MY_TABLE1");
        Select selectStatement = (Select) statement;
        TablesNamesFinder tablesNamesFinder = new TablesNamesFinder();
        List<String> tableList = tablesNamesFinder.getTableList(selectStatement);
```

## 应用别名

```java
// SELECT a AS A1, b AS A2, c AS A3 FROM test
Select select = (Select) CCJSqlParserUtil.parse("select a,b,c from test");
        final AddAliasesVisitor instance = new AddAliasesVisitor();
        select.getSelectBody().accept(instance);
```

## 添加一列或者表达式

```java
// SELECT a, b FROM mytable
Select select = (Select) CCJSqlParserUtil.parse("select a from mytable");
SelectUtils.addExpression(select, new Column("b"));
```

## 添加where语句

### 新建where

```java
Select select = (Select) CCJSqlParserUtil.parse("select name from user");
        PlainSelect plainSelect = (PlainSelect) select.getSelectBody();
        if (plainSelect.getWhere() == null) {
            EqualsTo equalsTo = new EqualsTo();
            equalsTo.setLeftExpression(new Column("id"));
            equalsTo.setRightExpression(new LongValue(1000L));
            plainSelect.setWhere(equalsTo);
        }
```

### 添加where

```java
Select select = (Select) CCJSqlParserUtil.parse("select name from user where id = 1000");
    PlainSelect plainSelect = (PlainSelect) select.getSelectBody();

    // 原where表达式
    Expression where = plainSelect.getWhere();
    // 新增的查询条件表达式
    EqualsTo equalsTo = new EqualsTo();
    equalsTo.setLeftExpression(new Column("name"));
    equalsTo.setRightExpression(new StringValue("'张三'"));
    // 用and链接条件
    AndExpression and = new AndExpression(where, equalsTo);
    // 设置新的where条件
    plainSelect.setWhere(and);
```

### 添加null

```java
Select select = (Select) CCJSqlParserUtil.parse("select name from user where id = 1000");
   PlainSelect plainSelect = (PlainSelect) select.getSelectBody();

   // 原where表达式
   Expression where = plainSelect.getWhere();
   // 新增的null判断条件
   IsNullExpression isNullExpression = new IsNullExpression();
   isNullExpression.setLeftExpression(new Column("name"));
   isNullExpression.setNot(true);
   // 用and链接条件
   AndExpression and = new AndExpression(where, isNullExpression);
   // 设置新的where条件
   plainSelect.setWhere(and);
```

# 生成

## 扩展插入

```java
// INSERT INTO mytable (col1) VALUES (1)
// INSERT INTO mytable (col1, col2) VALUES (1, 5)
// INSERT INTO mytable (col1, col2, col3) VALUES (1, 5, 10)

Insert insert = (Insert) CCJSqlParserUtil.parse("insert into mytable (col1) values (1)");
        System.out.println(insert.toString());
        insert.getColumns().add(new Column("col2"));
        insert.getItemsList().accept(new ItemsListVisitor() {

            public void visit(SubSelect subSelect) {
                throw new UnsupportedOperationException("Not supported yet.");
            }

            public void visit(ExpressionList expressionList) {
                expressionList.getExpressions().add(new LongValue(5));
            }

            public void visit(MultiExpressionList multiExprList) {
                throw new UnsupportedOperationException("Not supported yet.");
            }
        });
        System.out.println(insert.toString());
        insert.getColumns().add(new Column("col3"));
        ((ExpressionList) insert.getItemsList()).getExpressions().add(new LongValue(10));
```

## 建立select

```java
Select select = SelectUtils.buildSelectFromTable(new Table("mytable"));

Select select = SelectUtils.buildSelectFromTableAndExpressions(new Table("mytable"), new Column("a"), new Column("b"));

Select select = SelectUtils.buildSelectFromTableAndExpressions(new Table("mytable"), "a+b", "test");
```

## 代替字符串的值

```java
String sql ="SELECT NAME, ADDRESS, COL1 FROM USER WHERE SSN IN ('11111111111111', '22222222222222');";
Select select = (Select) CCJSqlParserUtil.parse(sql);

//Start of value modification
StringBuilder buffer = new StringBuilder();
ExpressionDeParser expressionDeParser = new ExpressionDeParser() {

    @Override
    public void visit(StringValue stringValue) {
	this.getBuffer().append("XXXX");
    }
    
};
SelectDeParser deparser = new SelectDeParser(expressionDeParser,buffer );
expressionDeParser.setSelectVisitor(deparser);
expressionDeParser.setBuffer(buffer);
select.getSelectBody().accept(deparser);
//End of value modification


System.out.println(buffer.toString());
//Result is: SELECT NAME, ADDRESS, COL1 FROM USER WHERE SSN IN (XXXX, XXXX)
import net.sf.jsqlparser.JSQLParserException;
import net.sf.jsqlparser.expression.LongValue;
import net.sf.jsqlparser.expression.StringValue;
import net.sf.jsqlparser.parser.CCJSqlParserUtil;
import net.sf.jsqlparser.statement.Statement;
import net.sf.jsqlparser.util.deparser.ExpressionDeParser;
import net.sf.jsqlparser.util.deparser.SelectDeParser;
import net.sf.jsqlparser.util.deparser.StatementDeParser;

public class ReplaceColumnValues {

    static class ReplaceColumnAndLongValues extends ExpressionDeParser {

        @Override
        public void visit(StringValue stringValue) {
            this.getBuffer().append("?");
        }

        @Override
        public void visit(LongValue longValue) {
            this.getBuffer().append("?");
        }
    }

    public static String cleanStatement(String sql) throws JSQLParserException {
        StringBuilder buffer = new StringBuilder();
        ExpressionDeParser expr = new ReplaceColumnAndLongValues();

        SelectDeParser selectDeparser = new SelectDeParser(expr, buffer);
        expr.setSelectVisitor(selectDeparser);
        expr.setBuffer(buffer);
        StatementDeParser stmtDeparser = new StatementDeParser(expr, selectDeparser, buffer);

        Statement stmt = CCJSqlParserUtil.parse(sql);

        stmt.accept(stmtDeparser);
        return stmtDeparser.getBuffer().toString();
    }

    public static void main(String[] args) throws JSQLParserException {
        System.out.println(cleanStatement("SELECT 'abc', 5 FROM mytable WHERE col='test'"));
        System.out.println(cleanStatement("UPDATE table1 A SET A.columna = 'XXX' WHERE A.cod_table = 'YYY'"));
        System.out.println(cleanStatement("INSERT INTO example (num, name, address, tel) VALUES (1, 'name', 'test ', '1234-1234')"));
        System.out.println(cleanStatement("DELETE FROM table1 where col=5 and col2=4"));
    }
}



/*
SELECT ?, ? FROM mytable WHERE col = ?
UPDATE table1 A SET A.columna = ? WHERE A.cod_table = ?
INSERT INTO example (num, name, address, tel) VALUES (?, ?, ?, ?)
DELETE FROM table1 WHERE col = ? AND col2 = ?
*/
```

# 参考

> https://github.com/JSQLParser/JSqlParser/wiki