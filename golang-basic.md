# Golang basic

Golang常见问题：

## 1.for range问题

```go
type Student struct {
	name string
	age  int
}

func main() {
	students := []Student{
		{"tom", 10},
		{"ken", 18},
	}

	for _, s := range students {	
		println(&s)	//实际上此处s的地址是不变的，for range从students中复制了一个副本到s中。
		println(s.name)	//此处是变化的
	}
}
```

