title: Go 中的nil channel

date: 2018-05-19 22:03:23

categories: Go

tags:

toc: true

---

### 从一个问题开始

要求写一个函数，该函数将从两个channel里获取值并将获取到的值放到一个新的channel里，最后将新的channel返回:
`func merge(a,b <-chan int) <-chan int`

### 一种思路

首先有一个生成a,b这两个channel的函数：

```go
func getChan(vs []int) <-chan int {
	c := make(chan int)
	go func(){
		for _, v := range vs{
			c <- v
			time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
		}
		close(c)
	}()
	return c
}
```

接着来实现merge函数，需要注意的是何时关闭channel：

```go
func merge(a, b <-chan int) <-chan int {
	c := make(chan int)
	go func() {
		defer close(c)
		aClosed, bClosed := false, false
		for !aClosed || !bClosed {
			select {
			case v, ok := <-a:
				if !ok {
					fmt.Println("channel a is closed")
					aClosed = true
					continue
				}
				c<-v
			case v, ok := <-b:
				if !ok {
					fmt.Println("channel b is closed")
					bClosed = true
					continue
				}
				c<-v
			}
		}
	}()
	return c
}
```

跑一下这个代码看到以下的结果：

```go
func main() {

	a := getChan([]int{1, 3, 5, 7})
	b := getChan([]int{2, 4, 6, 8})
	c := merge(a, b)
	for v := range c {
		fmt.Println(v)
	}
	fmt.Println("done")
}
```

```
>go run main.go
1
2
3
4
5
6
7
8
channel a is closed
channel a is closed
channel a is closed
channel a is closed
...
channel b is closed
done
```

然后会发现在`channel b close`之前输出了好多`channel a is closed `，原因是在两个channel都没有数据到来前select语句会阻塞在那里，但是一旦channel a 关闭了select则不会阻塞所以在channel b 数据到来之前select会一直选择channel a，此时进入了忙等待的状态。

### 另一种思路

如果一个channel close掉了，那么我们将这个channel置为nil看看效果会怎样。

```go
func merge(a, b <-chan int) <-chan int {
	c := make(chan int)
	go func() {
		defer close(c)
		for a!=nil || b!=nil {
			select {
			case v, ok := <-a:
				if !ok {
					fmt.Println("channel a is closed")
					a = nil
					continue
				}
				c <- v
			case v, ok := <-b:
				if !ok {
					fmt.Println("channel b is closed")
					b = nil
					continue
				}
				c <- v
			}
		}
	}()
	return c
}

```
执行main.go得到如下结果：

```go
1
3
2
5
4
6
7
8
channel a is closed
channel b is closed
done
```
这一次没有再出现疯狂输出`channel a is closed`，原因是一个值为nil的channel有以下特性：

* <-c 从channel中获取值的操作将一直被block
* ->c 向channel中放入值的操作将一直被block
* close(c)关闭一个值为nil的channel将产生panic

最后留一个问题，如何merge N 个channel，N是一个动态传入的参数。