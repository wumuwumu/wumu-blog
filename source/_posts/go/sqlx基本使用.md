---
title: sqlx基本使用
date: 2019-02-13 16:15:58
tags:
- go
---

# 安装

```bash
go get github.com/jmoiron/sqlx
```

# 连接数据库

```go
var db *sqlx.DB
 
// exactly the same as the built-in
db = sqlx.Open("sqlite3", ":memory:")
 
// from a pre-existing sql.DB; note the required driverName
db = sqlx.NewDb(sql.Open("sqlite3", ":memory:"), "sqlite3")
 
// force a connection and test that it worked
err = db.Ping()
```

# 查询

## Exec

直接执行，适合add,update,delete

```go
schema := `CREATE TABLE place (
    country text,
    city text NULL,
    telcode integer);`
 
// execute a query on the server
result, err := db.Exec(schema)
 
// or, you can use MustExec, which panics on error
cityState := `INSERT INTO place (country, telcode) VALUES (?, ?)`
countryCity := `INSERT INTO place (country, city, telcode) VALUES (?, ?, ?)`
db.MustExec(cityState, "Hong Kong", 852)
db.MustExec(cityState, "Singapore", 65)
db.MustExec(countryCity, "South Africa", "Johannesburg", 27)
```

## Query

查询数据库，适合select

```go
// fetch all places from the db
rows, err := db.Query("SELECT country, city, telcode FROM place")
 
// iterate over each row
for rows.Next() {
    var country string
    // note that city can be NULL, so we use the NullString type
    var city    sql.NullString
    var telcode int
    err = rows.Scan(&country, &city, &telcode)
}

// queryx 可以对结果转换成结构体
var person2 User
	rowxs,err :=db.Queryx("SELECT * FROM sys_user LIMIT 1")
	if err != nil{
		panic(err)
	}
	for rowxs.Next(){
		rowxs.StructScan(&person2)
		fmt.Println(person2)
	}
```

## Select

```go
p := Place{}
pp := []Place{}
 
// this will pull the first place directly into p
err = db.Get(&p, "SELECT * FROM place LIMIT 1")
 
// this will pull places with telcode > 50 into the slice pp
err = db.Select(&pp, "SELECT * FROM place WHERE telcode > ?", 50)
 
// they work with regular types as well
var id int
err = db.Get(&id, "SELECT count(*) FROM place")
 
// fetch at most 10 place names
var names []string
err = db.Select(&names, "SELECT name FROM place LIMIT 10")
```

# 事务

```go
// this will not work if connection pool > 1
db.MustExec("BEGIN;")
db.MustExec(...)
db.MustExec("COMMIT;")
```

# 预编译

```go
stmt, err := db.Prepare(`SELECT * FROM place WHERE telcode=?`)
row = stmt.QueryRow(65)
 
tx, err := db.Begin()
txStmt, err := tx.Prepare(`SELECT * FROM place WHERE telcode=?`)
row = txStmt.QueryRow(852)
```

# Named Queries

```go
// named query with a struct
p := Place{Country: "South Africa"}
rows, err := db.NamedQuery(`SELECT * FROM place WHERE country=:country`, p)
 
// named query with a map
m := map[string]interface{}{"city": "Johannesburg"}
result, err := db.NamedExec(`SELECT * FROM place WHERE city=:city`, m)


p := Place{TelephoneCode: 50}
pp := []Place{}
 
// select all telcodes > 50
nstmt, err := db.PrepareNamed(`SELECT * FROM place WHERE telcode > :telcode`)
err = nstmt.Select(&pp, p)


arg := map[string]interface{}{
    "published": true,
    "authors": []{8, 19, 32, 44},
}
query, args, err := sqlx.Named("SELECT * FROM articles WHERE published=:published AND author_id IN (:authors)", arg)
query, args, err := sqlx.In(query, args...)
query = db.Rebind(query)
db.Query(query, args...)
```



# 参考

> http://jmoiron.github.io/sqlx/

