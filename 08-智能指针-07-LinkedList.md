## `LinkedList<T>`代码分析

双向链表及其他数据结构的代码实现都是经典的实用性及训练性上佳的项目。本书对这些经典数据结构将只分析LinkedList<T>，重点分析RUST与其他语言的不同的部分。如果对LinkedList<T>彻底理解了，那其他数据结构也就不成为问题:
`LinkedList<T>`类型结构定义如下：
```rust
//这个定义表示LinkedList只支持固定长度的T类型
pub struct LinkedList<T> {
    //等同于直接用裸指针，使得代码最方便及简化，但需要对安全性额外投入精力
    //这个实际上与C语言相同，只是用Option增加了安全措施
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    //marker说明本结构有一个Box<Node<T>>的所有权，并会负责调用其的drop
    //编译器应做好drop check, 检查与本结构相关的Box<Node<T>>的生命周期及drop
    //marker体现了RUST的独特点
    marker: PhantomData<Box<Node<T>>>,
}

struct Node<T> {
    next: Option<NonNull<Node<T>>>,
    prev: Option<NonNull<Node<T>>>,
    element: T,
}
```
Node方法代码：
```rust
impl<T> Node<T> {
    fn new(element: T) -> Self {
        Node { next: None, prev: None, element }
    }

    fn into_element(self: Box<Self>) -> T {
        //消费了Box，堆内存被释放并将element拷贝到栈
        self.element
    }
}
```
LinkedList的创建及简单的增减方法：
```rust
impl<T> LinkedList<T> {
    //创建一个空的LinkedList
    pub const fn new() -> Self {
        LinkedList { head: None, tail: None, len: 0, marker: PhantomData }
    }
```
在头部增加一个成员及删除一个成员：
```rust
    //在首部增加一个节点
    pub fn push_front(&mut self, elt: T) {
        //用box从堆内存申请一个节点，push_front_node见后面函数
        self.push_front_node(box Node::new(elt));
    }
    fn push_front_node(&mut self, mut node: Box<Node<T>>) {
        // 整体全是不安全代码
        unsafe {
            node.next = self.head;
            node.prev = None;
            //需要将Box的堆内存leak出来使用。此块内存后继如果还在链表，需要由LinkedList负责drop.后面可以看到LinkedList的drop函数的处理。
            //如果pop出链表，那会重新用这里leak出来的NonNull<Node<T>>生成Box,再由Box释放
            let node = Some(Box::leak(node).into());

            match self.head {
                //空链表
                None => self.tail = node,
                // 目前采用NonNull<Node<T>>的方案，此处代码就很自然
                // 如果换成Box<Node<T>>的方案，这里就要类似如下:
                // 先用take将head复制到栈中创建的新变量，
                // 新变量的prev置为node
                // 用replace将新变量再复制回head。
                // 也注意，此处很容易也采用先take, 修改，然后replace的方案
                // 要注意规避Option导致的这个习惯,会造成两次内存拷贝，效率太低
                Some(head) => (*head.as_ptr()).prev = node,
            }

            self.head = node;
            self.len += 1;
        }
    }

    //从链表头部删除一个节点
    pub fn pop_front(&mut self) -> Option<T> {
        //Option<T>::map，此函数后，节点的堆内存已经被释放
        //变量被拷贝到栈内存
        self.pop_front_node().map(Node::into_element)
    }
    fn pop_front_node(&mut self) -> Option<Box<Node<T>>> {
        //整体是unsafe
        self.head.map(|node| unsafe {
            //重新生成Box，以便后继可以释放堆内存
            let node = Box::from_raw(node.as_ptr());
            //更换head指针
            self.head = node.next;

            match self.head {
                None => self.tail = None,
                // push_front_node() 已经分析过
                Some(head) => (*head.as_ptr()).prev = None,
            }

            self.len -= 1;
            node
        })
    }
```
在尾部增加一个成员及删除一个成员
```rust
    //从尾部增加一个节点
    pub fn push_back(&mut self, elt: T) {
        //用box从堆内存申请一个节点
        self.push_back_node(box Node::new(elt));
    }

    fn push_back_node(&mut self, mut node: Box<Node<T>>) {
        // 整体不安全
        unsafe {
            node.next = None;
            node.prev = self.tail;
            //需要将Box的堆内存leak出来使用。此块内存后继如果还在链表，需要由LinkedList负责drop.
            //如果pop出链表，那会重新用这里leak出来的NonNull<Node<T>>重新生成Box
            let node = Some(Box::leak(node).into());

            match self.tail {
                None => self.head = node,
                //前面代码已经有分析
                Some(tail) => (*tail.as_ptr()).next = node,
            }

            self.tail = node;
            self.len += 1;
        }
    }

    //从尾端删除节点
    pub fn pop_back(&mut self) -> Option<T> {
        self.pop_back_node().map(Node::into_element)
    }

    fn pop_back_node(&mut self) -> Option<Box<Node<T>>> {
        self.tail.map(|node| unsafe {
            //重新创建Box以便删除堆内存
            let node = Box::from_raw(node.as_ptr());
            self.tail = node.prev;

            match self.tail {
                None => self.head = None,

                Some(tail) => (*tail.as_ptr()).next = None,
            }

            self.len -= 1;
            node
        })
    }
    //删除一个节点，这个操作也是RUST比较独特的体现
    unsafe fn unlink_node(&mut self, mut node: NonNull<Node<T>>) {
        //现在拥有node的所有权，
        let node = unsafe { node.as_mut() };

        match node.prev {
            //不能复制新的节点，注意这里的写法
            Some(prev) => unsafe { (*prev.as_ptr()).next = node.next },
            // node是head节点
            None => self.head = node.next,
        };

        match node.next {
            //不能获取next的所有权，只能是这个写法
            Some(next) => unsafe { (*next.as_ptr()).prev = node.prev },
            // node是tail节点
            None => self.tail = node.prev,
        };

        self.len -= 1;
    }
    ...
}

//Drop
unsafe impl<#[may_dangle] T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        struct DropGuard<'a, T>(&'a mut LinkedList<T>);

        impl<'a, T> Drop for DropGuard<'a, T> {
            fn drop(&mut self) {
                //如果此函数后面的while循环出现panic，这里可以继续做释放
                //此处代码的存在应该是RUST标准库中隐藏比较深的bug导致
                while self.0.pop_front_node().is_some() {}
            }
        }

        while let Some(node) = self.pop_front_node() {
            let guard = DropGuard(self);
            //显式的drop 获取的Box<Node<T>>
            drop(node);
            //执行到此处，guard认为已经完成，不能再调用guard的drop
            mem::forget(guard);
        }
    }
}
```
以上基本上说明了RUST的LinkedList的设计及代码的一些关键点。

用Iterator来对List进行访问，Iterator的相关结构代码如下：
into_iter()相关结构及方法：
```rust
//变量本身的Iterator的类型
pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> IntoIterator for LinkedList<T> {
    type Item = T;
    type IntoIter = IntoIter<T>;

    /// 对LinkedList<T> 做消费
    fn into_iter(self) -> IntoIter<T> {
        IntoIter { list: self }
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<T> {
        //从头部获取变量
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}
```
iter_mut()调用相关结构及方法
```rust
//可变引用的Iterator的类型
pub struct IterMut<'a, T: 'a> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    //这个marker也标示了IterMut对LinkedList有一个可变引用
    //创建IterMut后，与之相关的LinkerList不能在被其他安全的代码修改
    marker: PhantomData<&'a mut Node<T>>,
}

impl <T> LinkedList<T> {
    ...
    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { head: self.head, tail: self.tail, len: self.len, marker: PhantomData }
    }
    ...
}
impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<&'a mut T> {
        if self.len == 0 {
            None
        } else {
            //用Option::map简化代码
            self.head.map(|node| unsafe {
                // 保存首部成员
                let node = &mut *node.as_ptr();
                // 删除首部成员
                self.len -= 1;
                self.head = node.next;
                // 返回可变引用，此处的生命周期如下：
                // 返回值生命周期小于self
                // self 生命周期小于 LinkedList
                &mut node.element
            })
        }
    }

    ...
}

//不可变引用的Iterator的类型
pub struct Iter<'a, T: 'a> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    //对生命周期做标识，也标识了一个对LinkedList的不可变引用
    marker: PhantomData<&'a Node<T>>,
}

impl<T> Clone for Iter<'_, T> {
    fn clone(&self) -> Self {
        //本书中第一次出现这个表述
        Iter { ..*self }
    }
}

//Iterator trait for Iter略
```
LinkedList其他的代码略。
LinkedList当然有不使用unsafe方式的实现方法，但是unsafe的实现方式最简化，效率最高。且unsafe的代码量并不高，可控性很强。盲目的排斥unsafe实际上也是一件不RUST的事。

