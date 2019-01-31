# Golang Map和Slice获取效率

当数据的长度较短（小于30或更小）的时候，对Slice进行遍历的效率比从Map获取更高，使用内存空间也更少。但是当数据长度较长的情况下，Map的优势就更大了。

用以下代码进行基准测试获得的结果如下：

* 数据长度2：

```text
BenchmarkMapGet-4       100000000               11.0 ns/op
BenchmarkSliceGet-4     300000000                5.19 ns/op
```

* 数据长度5：

```text
BenchmarkMapGet-4       100000000               11.0 ns/op
BenchmarkSliceGet-4     200000000                7.05 ns/op
```

* 数据长度10：

```text
BenchmarkMapGet-4       100000000               17.5 ns/op
BenchmarkSliceGet-4     100000000               10.5 ns/op
```

* 数据长度30：

```text
BenchmarkMapGet-4       100000000               18.5 ns/op
BenchmarkSliceGet-4     100000000               21.2 ns/op
```

* 数据长度100：

```text
BenchmarkMapGet-4       100000000               24.3 ns/op
BenchmarkSliceGet-4     20000000                63.4 ns/op
```

```go
package main

import (
	"testing"
	"strconv"
	"fmt"
)

const DATA_LENGTH = 100

type methodTree struct {
	method string
	root   *string
}

func mapGet(trees map[string]*string, method string) *string {
	if result := trees[method]; result != nil {
		return result
	}
	return nil
}

func sliceGet(trees []methodTree, method string) *string {
	for _, tree := range trees {
		if tree.method == method {
			return tree.root
		}
	}
	return nil
}

func BenchmarkMapGet(b *testing.B) {
	trees, keys := getMapDataAndKeys();
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		mapGet(trees, keys[i%DATA_LENGTH])
	}
}
func getMapDataAndKeys() (map[string]*string, []string) {
	trees := make(map[string]*string)
	str := ""
	for i := 0; i < DATA_LENGTH; i++ {
		trees[strconv.Itoa(i)] = &str
	}

	keys := make([]string, DATA_LENGTH)
	for i := 0; i < DATA_LENGTH; i++ {
		keys = append(keys, strconv.Itoa(i))
	}
	return trees, keys
}

func BenchmarkSliceGet(b *testing.B) {
	trees, keys := getSliceDataAndKeys()
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		sliceGet(trees, keys[i%DATA_LENGTH])
	}
}
func getSliceDataAndKeys() ([]methodTree, []string) {
	trees := make([]methodTree, 0)
	str := ""
	for i := 0; i < DATA_LENGTH; i++ {
		trees = append(trees, methodTree{method: strconv.Itoa(i), root: &str})
	}
	keys := make([]string, DATA_LENGTH)
	for i := 0; i < DATA_LENGTH; i++ {
		keys = append(keys, strconv.Itoa(i))
	}
	return trees, keys
}

func main() {
	fmt.Println(getSliceDataAndKeys())
	fmt.Println(getMapDataAndKeys())
}

```

