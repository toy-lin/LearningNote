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

## 2.协程切换问题

如果一个正在运行的协程没有执行耗时操作，那么它就不会被挂起。

```go
func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	times := 1000
	wg.Add(times)
	for i := 0; i < times; i++ {
		go func() {
			fmt.Println("A: ", i)	//此处输出永远是times的值，即1000。
			wg.Done()
		}()
	}
	wg.Wait()
}
```

