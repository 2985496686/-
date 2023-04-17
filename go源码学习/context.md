# 结构体内嵌匿名接口

go中的context除emptyContext外，都内嵌了Context接口，并且超级父类emptyContext实现了接口的全部方法。
- 与内嵌匿名结构体不同，需要初始化匿名的接口字段(为它赋值实现类)，才能调用方法，不然会报空指针。
- 如果有多层会调用离他最近的实现。
```go
type node interface {  
   print()  
}  
  
type node1 struct {  
   node  
   x int  
}  
  
func (n *node1) print() {  
   fmt.Println(n.x)  
}  
  
type node2 struct {  
   node  
   y int  
}  
  
type node3 struct {  
   node  
}  
  
func TestInterface(t *testing.T) {  
   n1 := node1{x: 1}  
   n2 := node2{node: &n1, y: 2}  
   n3 := node3{node: &n2}  
   n3.print()  
}
```
node2没有实现该方法，所以会调用node1的实现。


# Context

```go
type Context interface {
	 Deadline() (deadline time.Time, ok bool)
	 Done() <-chan struct{}
	 Err() error
	 Value(key interface{}) interface{}
}

var (  
   background = new(emptyCtx)  
   todo       = new(emptyCtx)  
)  
 
func Background() Context {  
   return background  
}  

 
func TODO() Context {  
   return todo  
}

```

go语言中上下文对象的创建都要基于emptyCtx，不能直接创建。



# WithCancelContext

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {  
   if parent == nil {  
      panic("cannot create context from nil parent")  
   }  
   c := newCancelCtx(parent)  
   propagateCancel(parent, &c)  
   return &c, func() { c.cancel(true, Canceled) }  
}
```
**下面是WithCancelContext的核心代码**

```go
//将当前上下文挂载到父上下文
func propagateCancel(parent Context, child canceler) {  
   done := parent.Done()  
   // 1. 父节点中没有可以cancel的上下文，父节点中只有emptyContext和valueContext(除用户自己声明的context结构体外)
   if done == nil {  
      return
   }
  //2. 父节点已经被cancel，直接cancel当前节点
   select {  
   case <-done:  
      // parent is already canceled  
      child.cancel(false, parent.Err())  
      return  
   default:  
   }  
  //3. 寻找该上下文能够挂载的父节点
   if p, ok := parentCancelCtx(parent); ok {  
      p.mu.Lock()  
      if p.err != nil {  
         // 父节点已经被cancel 
         child.cancel(false, p.err)  
      } else {  
	      //挂载当前上下文
         if p.children == nil {  
            p.children = make(map[canceler]struct{})  
         }  
         p.children[child] = struct{}{}  
      }  
      p.mu.Unlock()  
   } else {  
	   //parentCancelCtx返回false，说明没有
      atomic.AddInt32(&goroutines, +1)  
      go func() {  
         select {  
         case <-parent.Done():  
            child.cancel(false, parent.Err())  
         case <-child.Done():  
         }  
      }()  
   }  
}
```
**说明**
- 第一点： emptyContext的Done方法会返回nil，withValue的Done方法没有实现，所以说只有emptyContext和withValueContext的父节点会返回nil。
- 第三点：如果用户自己实现了一个context，并且实现Done方法返回一个非空的chan，这样的上下文虽然能够cancel，但是无法挂载子节点，所以会开启一个携程来监听父节点的cancel。



```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {  
   done := parent.Done()  
   if done == closedchan || done == nil {  
      return nil, false  
   }  
   p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)  
   if !ok {  
      return nil, false  
   }  
   pdone, _ := p.done.Load().(chan struct{})  
   if pdone != done {  
      return nil, false  
   }  
   return p, true  
}
```


这段代码主要是：`` p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)  `` ，能够挂载的父节点(可以被cancel的)，有两种情况: 
1. 当前节点是cancelContext节点，调用Value方法如下：
```go
func (c *cancelCtx) Value(key any) any {  
   if key == &cancelCtxKey {  
      return c  
   }  
   return value(c.Context, key)  
}
```
如果当前节点为可以cancel的，直接返回i。

2. 当前节点是valueContext节点，调用如下：
```go
func (c *valueCtx) Value(key any) any {  
   if c.key == key {  
      return c.val  
   }  
   return value(c.Context, key)  
}  
 
```
这两种情都会调用同一个方法
```go
func value(c Context, key any) any {  
   for {  
      switch ctx := c.(type) {  
      case *valueCtx:  
         if key == ctx.key {  
            return ctx.val  
         }  
         c = ctx.Context  
      case *cancelCtx:  
         if key == &cancelCtxKey {  
            return c  
         }  
         c = ctx.Context  
      case *timerCtx:  
         if key == &cancelCtxKey {  
            return &ctx.cancelCtx  
         }  
         c = ctx.Context  
      case *emptyCtx:  
         return nil  
      default:  
         return c.Value(key)  
      }  
   }  
}
```

递归找到可以取消的父类。

这块的设计十分巧妙，为cancelCtx定义了一个key，从而实现了方法的复用。

**cancel方法**

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    // 必须要传 err
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // 已经被其他协程取消
	}
	// 给 err 字段赋值
	c.err = err
	// 关闭 channel，通知其他协程
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	// 遍历它的所有子节点
	for child := range c.children {
	    // 递归地取消所有子节点
		child.cancel(false, err)
	}
	// 将子节点置空
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
	    // 从父节点中移除自己 
		removeChild(c.Context, c)
	}
}
```

## withTimeout和withDeadline

withTimeout 其实就是withDeadline

withDeadline只是比withCancel增加了一个定时器。





<!--stackedit_data:
eyJoaXN0b3J5IjpbMjExMTMyMTMwMl19
-->