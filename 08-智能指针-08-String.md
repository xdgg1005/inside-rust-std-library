## String 类型分析

String结构定义如下：
```rust
pub struct String {
    vec: Vec<u8>,
}
```
`Vec<u8>`和String的关系可以与[u8]与&str的关系相对比。整个String实际上是一个大的
Adapter 模式，针对Vec<u8>, [u8], &str三者做组合
String的创建函数：
```rust
impl String {
     pub const fn new() -> String {
        String { vec: Vec::new() }
    }

    //将str内容加到String的尾部
    pub fn push_str(&mut self, string: &str) {
        //adapter，直接用Vec::extend_from_slice([u8])
        //具体的细节请参考Vec那节
        self.vec.extend_from_slice(string.as_bytes())
    }
    ...
}

impl ToOwned for str {
    type Owned = String;
    fn to_owned(&self) -> String {
        //这里是个adapter模式，首先从用self.as_bytes()获取[u8], 然后用通用的[u8].to_owned()完成
        //to_owned逻辑,随后从Vec[u8]生成String
        unsafe { String::from_utf8_unchecked(self.as_bytes().to_owned()) }
    }


    fn clone_into(&self, target: &mut String) {
        //adapter模式，需要先得到Vec<u8>,因为into_bytes会消费掉String。
        //target不支持，所以需要用take先把所有权转移出来，然后获取Vec<u8>
        //这是RUST的一个通用的技巧
        let mut b = mem::take(target).into_bytes();
        //通用的[u8].clone_into
        self.as_bytes().clone_into(&mut b);
        //把新的String赋给原先的地址
        *target = unsafe { String::from_utf8_unchecked(b) }
    }
}

impl From<&str> for String {
    fn from(s: &str) -> String {
        s.to_owned()
    }
}


```
解引用方法代码：
```rust
impl ops::Deref for String {
    type Target = str;

    fn deref(&self) -> &str {
        //&self.vec会被强转为&[u8]
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}

impl ops::DerefMut for String {
    fn deref_mut(&mut self) -> &mut str {
        //这里直接用&mut self.vec应该也可以，会被强转成&mut [u8]
        unsafe { str::from_utf8_unchecked_mut(&mut *self.vec) }
    }
}
```
运算符重载方法
```rust
impl ops::Index<ops::RangeFull> for String {
    type Output = str;

    fn index(&self, _index: ops::RangeFull) -> &str {
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}
impl ops::Index<ops::Range<usize>> for String {
    type Output = str;

    fn index(&self, index: ops::Range<usize>) -> &str {
        //先用Index<RangeFull>取&str, 然后用Index<Range>取子串
        &self[..][index]
    }
}
impl Borrow<str> for String {
    fn borrow(&self) -> &str {
        //自动解引用, 利用Index<RangeFull>完成，代码最简
        &self[..]
    }
}

impl BorrowMut<str> for String {
    fn borrow_mut(&mut self) -> &mut str {
        //自动解引用, 利用Index<RangeFull>完成，代码最简
        &mut self[..]
    }
}

impl Add<&str> for String {
    type Output = String;

    fn add(mut self, other: &str) -> String {
        self.push_str(other);
        self
    }
}

impl AddAssign<&str> for String {
    fn add_assign(&mut self, other: &str) {
        self.push_str(other);
    }
}
```
字符串数组连接方法：
```rust
//此函数主要简化多个字符串的连接
impl<S: Borrow<str>> Concat<str> for [S] {
    type Output = String;

    fn concat(slice: &Self) -> String {
        //见下个方法分析
        Join::join(slice, "")
    }
}

impl<S: Borrow<str>> Join<&str> for [S] {
    type Output = String;

    fn join(slice: &Self, sep: &str) -> String {
        unsafe { String::from_utf8_unchecked(join_generic_copy(slice, sep.as_bytes())) }
    }
}

macro_rules! specialize_for_lengths {
    ($separator:expr, $target:expr, $iter:expr; $($num:expr),*) => {{
        let mut target = $target;
        let iter = $iter;
        let sep_bytes = $separator;
        match $separator.len() {
            $(
                // 如果分隔切片长度符合预设值
                $num => {
                    for s in iter {
                        //拷贝分隔切片到目的切片，且更新目的切片
                        copy_slice_and_advance!(target, sep_bytes);
                        //拷贝内容切片
                        let content_bytes = s.borrow().as_ref();
                        copy_slice_and_advance!(target, content_bytes);
                    }
                },
            )*
            _ => {
                // 如果分隔切片长度不符合预设值，实质也做与上段代码同样的操作
                for s in iter {
                    copy_slice_and_advance!(target, sep_bytes);
                    let content_bytes = s.borrow().as_ref();
                    copy_slice_and_advance!(target, content_bytes);
                }
            }
        }
        target
    }}
}

//完成一个切片拷贝后，切片向前到未拷贝的开始处
macro_rules! copy_slice_and_advance {
    ($target:expr, $bytes:expr) => {
        let len = $bytes.len();
        //将目的切片切分成两段，首段为待拷贝空间，尾端为未拷贝空间
        let (head, tail) = { $target }.split_at_mut(len);
        head.copy_from_slice($bytes);
        $target = tail;
    };
}

//将若干个T类型的切片连接到一起形成一个基于T类型的切片
fn join_generic_copy<B, T, S>(slice: &[S], sep: &[T]) -> Vec<T>
where
    T: Copy, //最基础的成员类型
    B: AsRef<[T]> + ?Sized, //可以表示为最基础成员的切片引用
    S: Borrow<B>, //以B类型作为操作类型，所以S应该能borrow成B类型的引用
{
    let sep_len = sep.len();
    let mut iter = slice.iter();

    // 第一个成员头部没有间隔
    let first = match iter.next() {
        Some(first) => first,
        None => return vec![],
    };

    //计算iter中所有成员的长度，并加上间隔长度乘剩余成员的数目
    //得到总的长度。
    //从这个函数能够发现rust的链式编程的能力

    let reserved_len = sep_len
        .checked_mul(iter.len())//这里去掉了slice的首个成员，
        .and_then(|n| {
            //这里的重新重新生成iter，计算了所有的slice的所有成员
            slice.iter().map(|s| s.borrow().as_ref().len()).try_fold(n, usize::checked_add)
        })
        .expect("attempt to join into collection with len > usize::MAX");

    // 创建一个有足够容量的Vec
    let mut result = Vec::with_capacity(reserved_len);
    debug_assert!(result.capacity() >= reserved_len);
    //完成first的内容拷贝
    result.extend_from_slice(first.borrow().as_ref());

    unsafe {
        let pos = result.len();
        let target = result.get_unchecked_mut(pos..reserved_len);

        //完成对剩余成员及分隔符拷贝到result
        let remain = specialize_for_lengths!(sep, target, iter; 0, 1, 2, 3, 4);

        //完成长度拷贝。
        let result_len = reserved_len - remain.len();
        result.set_len(result_len);
    }
    result
}
```
