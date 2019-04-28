---
title: go基本语法
date: 2019-04-10 10:29:55
tags:
- go
---

# 接口

1. duck typing了解

在[程序设计](https://zh.wikipedia.org/wiki/%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)中，**鸭子类型**（英语：**duck typing**）是[动态类型](https://zh.wikipedia.org/wiki/%E9%A1%9E%E5%9E%8B%E7%B3%BB%E7%B5%B1)的一种风格。在这种风格中，一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由“当前[方法](https://zh.wikipedia.org/wiki/%E6%96%B9%E6%B3%95_(%E9%9B%BB%E8%85%A6%E7%A7%91%E5%AD%B8))和属性的集合”决定。

# flag

# Sync

### 1. `WaitGroup`

```
Add()
Done()
Wait()
```

### 2. Context

```

```

## `Regexp`

> https://www.cnblogs.com/golove/p/3269099.html

```
// MatchString
matched, err := regexp.MatchString("foo.*", "seafood")
fmt.Println(matched, err)
matched, err = regexp.MatchString("bar.*", "seafood")
fmt.Println(matched, err)
// false error parsing regexp: missing closing ): `a(b`
matched, err = regexp.MatchString("a(b", "seafood")
fmt.Println(matched, err)
// true <nil>
matched, err = regexp.MatchString(`a\(b`, "a(b")
fmt.Println(matched, err)
// false error parsing regexp: missing closing ): `a(b`
matched, err = regexp.MatchString(`a(b`, "a(b")
fmt.Println(matched, err)
// true <nil>
matched, err = regexp.MatchString("a\\(b", "a(b")
fmt.Println(matched, err)


// 将所有特殊字符进行转义
fmt.Println(regexp.QuoteMeta("Escaping symbols like: .+*?()|[]{}^$"))


// ExpandString
content := `
	# comment line
	option1: value1
	option2: value2

	# another comment line
	option3: value3
`

// Regex pattern captures "key: value" pair from the content.
pattern := regexp.MustCompile(`(?m)(?P<key>\w+):\s+(?P<value>\w+)$`)

// Template to convert "key: value" to "key=value" by
// referencing the values captured by the regex pattern.
template := "$key=$value\n"

result := []byte{}

	// For each match of the regex in the content.
for _, submatches := range pattern.FindAllStringSubmatchIndex(content, -1) {
    // Apply the captured submatches to the template and append the output
    // to the result.
    result = pattern.ExpandString(result, template, content, submatches)
}
fmt.Println(string(result))


// findAllString
re := regexp.MustCompile("a.")
fmt.Println(re.FindAllString("paranormal", -1))
fmt.Println(re.FindAllString("paranormal", 2))
fmt.Println(re.FindAllString("graal", -1))
fmt.Println(re.FindAllString("none", -1))


// FindAllStringSubmatch
re := regexp.MustCompile("a(x*)b")
fmt.Printf("%q\n", re.FindAllStringSubmatch("-ab-", -1))
fmt.Printf("%q\n", re.FindAllStringSubmatch("-axxb-", -1))
fmt.Printf("%q\n", re.FindAllStringSubmatch("-ab-axb-", -1))
fmt.Printf("%q\n", re.FindAllStringSubmatch("-axxb-ab-", -1))

// findStringSubmatch，只查找第一个
re := regexp.MustCompile("a(x*)b(y|z)c")
fmt.Printf("%q\n", re.FindStringSubmatch("-axxxbyc-"))
fmt.Printf("%q\n", re.FindStringSubmatch("-abzc-"))
```