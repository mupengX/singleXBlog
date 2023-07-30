title: 一场因 json null 搭配 interface 使用而引发的 panic

date: 2023-07-30 11:06:13

categories: Go

tags:

---

上周因同事休假暂时帮忙维护一个服务，周五下午本以为要顺利度过本周的时候线上出了问题，在进行数据修复的时候发现之前一个工具代码的隐藏 bug，该 bug 会在进行数据修复时触发 panic。剥离具体业务逻辑之后代码逻辑如下：

```golang
func main() {
	// 从数据库中查询数据，a 为具体数据的指针，当数据没找到时 a=nil
	a := findFromDB()
	m, _ := StructToMap(a)
	// 通过 builder 构建数据
	b := NewJSONBuilder()
	b.SetData(m)
	// 省略部分逻辑
	....
	b.AddValueWithPath(mask.DstPath, v)
	....
}


```
`main` 函数逻辑很简单：从数据库中查询某个数据，将该数据转换为`map[string]interface{}`，再通过 `JsonBuilder` 根据`JsonPath`对数据进行修改，其中依赖的其他函数如下：

```golang
func StructToMap(v interface{}) (map[string]interface{}, error) {
	bytes, err := json.Marshal(v)
	if err != nil {
		return nil, err
	}
	var m map[string]interface{}
	err = json.Unmarshal(bytes, &m)
	if err != nil {
		return nil, err
	}
	return m, nil
}

type JSONBuilder struct {
	data interface{}
}

func (b *JSONBuilder) SetData(data interface{}) error {
	if b == nil {
		return ErrNilBuilder
	}
	b.data = data
	return nil
}

func (b *JSONBuilder) AddValueWithPath(path string, value interface{}) (*JSONBuilder, error) {
	// 省略部分逻辑
	....
	if b.data == nil {
		b.data = make(map[string]interface{})
	}
	data = b.data
	for i:=0; i<len(keys); i++ {
		// 省略部分逻辑
		....
		
		data[keys[i]]=value // panic！
		
		// 省略部分逻辑
		.....
	
	}
	....

}

```

在讨论为何会引发 panic 之前，有几个背景知识需要交代清楚：

1.  `json.Marshal(v any) ([]byte, error)` 函数如果入参为 nil 则返回的 bytes 内容为`null`
2. `json.Unmarshal(data []byte, v any)` 函数如果入参 `data`为`null`则函数执行结束之后 V 的值为 nil
3. golang 的 interface 的内部实现两个字段：`type` 和 `data`，只有两个字段都为 nil 时`interface == nil`才为 `true`

这里引发 panic 的主要原因是`StructToMap`返回的 map 为 nil，在后续将其赋值给了`builder.data`， 而 `data` 为`interface` 类型，因此该`interface`的`type`为`map[string]interface{}`，而该`interface`的`data`为 `nil`，最终导致`if b.data == nil `的判断语句失效，进而没有进行数据初始化引发了 panic。

这里只需将`if b.data == nil `修改为`if b.data == nil || reflect.ValueOf(b.data).IsNil()` 即可准确的判断出 `interface`是否为 nil，从而完成数据初始化避免 panic


golang 中的 inteface 是否为 nil 的判断已经是老生常谈的问题了，但在实际开发过程中还是经常被忽略掉，再搭配到 json 包一起使用，一不小心还是很容易踩坑。