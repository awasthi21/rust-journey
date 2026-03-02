# Pointers in Rust — Complete Guide (Hinglish Q&A)

## Pointer Types Overview

| # | Type | Symbol | Safe? |
|---|------|---------|-------|
| 1 | References | `&T` / `&mut T` | Yes |
| 2 | Raw Pointers | `*const T` / `*mut T` | No (unsafe) |
| 3 | Box | `Box<T>` | Yes |
| 4 | Rc | `Rc<T>` | Yes |
| 5 | Arc | `Arc<T>` | Yes |
| 6 | Weak | `Weak<T>` | Yes |
| 7 | RefCell | `RefCell<T>` | Yes (runtime check) |

---

# 1. References

## Q1. Reference kya hoti hai?
Reference ek safe pointer hai jo kisi value ko borrow karta hai — ownership nahi leta.

```rust
fn main() {
    let x = 10;
    let p = &x;
    println!("{}", *p);  // 10
}
```

## Q2. Immutable vs Mutable Reference?

```rust
fn main() {
    let x = 10;
    let p = &x;          // Immutable ref — sirf read
    // *p = 20;          // ERROR

    let mut y = 10;
    let q = &mut y;      // Mutable ref — read + write
    *q = 20;
    println!("{}", y);   // 20
}
```

## Q3. Golden Rule?

```
Ya toh -> unlimited immutable references (&T)
Ya toh -> sirf 1 mutable reference (&mut T)
DONO SAATH NAHI
```

```rust
fn main() {
    let mut x = 10;
    let a = &x;
    let b = &x;       // OK — multiple immutable allowed
    println!("{} {}", a, b);

    let c = &mut x;   // OK (a,b ab use nahi ho rahe)
    *c = 99;
}
```

## Q4. Function mein reference pass karna?

```rust
fn print_val(p: &i32) {
    println!("Value: {}", p);
}

fn add_one(p: &mut i32) {
    *p += 1;
}

fn main() {
    let x = 10;
    print_val(&x);
    println!("{}", x);  // x still valid!

    let mut y = 5;
    add_one(&mut y);
    println!("{}", y);  // 6
}
```

## Q5. Dangling Reference?

```rust
fn dangle() -> &i32 {  // COMPILE ERROR!
    let x = 10;
    &x   // x yahan destroy ho jayega!
}
```

Rust ka borrow checker compile time pe hi pakad leta hai.

## Q6. Lifetime intro?

```rust
fn longer<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }
}
```

## Q7. Reference ka data kahan hota hai? (Important Clarification)

Reference data store nahi karta — sirf address store karta hai!
Data kahan hai? Depends on original variable.

```
Case 1: Stack data
let x = 5;     // x -> STACK
let r = &x;    // r -> stack ko point karta hai

Case 2: Heap data
let s = String::from("hello");  // data -> HEAP
let r = &s;                     // r -> heap ko point karta hai

Case 3: &str slice
let s = String::from("hello world");
let slice: &str = &s[0..5];
// slice = fat pointer (address + length), data heap pe
```

Key insight: &T apna data choose nahi karta.
Box<T> hamesha khud heap pe data rakhta hai.

## References Summary

| | `&T` | `&mut T` |
|--|------|----------|
| Read | Yes | Yes |
| Write | No | Yes |
| Ek saath kitne | Unlimited | Sirf 1 |
| Ownership | Nahi leti | Nahi leti |
| Dangling possible? | Never | Never |

---

# 2. Raw Pointers (`*const T` / `*mut T`)

## Q1. Raw Pointer kya hota hai?
C/C++ ke pointer jaisa — koi safety guarantee nahi, compiler kuch check nahi karta.

```rust
fn main() {
    let x = 10;
    let p = &x as *const i32;
    println!("{:p}", p);
}
```

## Q2. Do types hain:

```rust
*const T   // immutable (C ka const int*)
*mut T     // mutable   (C ka int*)
```

## Q3. Dereference karna?

Sirf unsafe block ke andar!

```rust
fn main() {
    let x = 10;
    let p = &x as *const i32;
    unsafe {
        println!("{}", *p);  // 10
    }
}
```

## Q4. Banane ke tarike?

```rust
// Method 1: cast
let p1 = &x as *const i32;
let p2 = &mut y as *mut i32;

// Method 2: addr_of! macro (recommended)
let p3 = std::ptr::addr_of!(x);
let p4 = std::ptr::addr_of_mut!(y);
```

## Q5. Write karna?

```rust
let mut x = 10;
let p = &mut x as *mut i32;
unsafe { *p = 99; }
println!("{}", x);  // 99
```

## Q6. Null Pointer?

```rust
let p: *const i32 = std::ptr::null();
if p.is_null() {
    println!("Null hai!");
}
```

## Q7. Pointer Arithmetic?

```rust
let arr = [10, 20, 30, 40, 50];
let p = arr.as_ptr();
unsafe {
    println!("{}", *p.add(1));    // 20
    println!("{}", *p.offset(2)); // 30
}
```

## Q8. References vs Raw Pointers

| | `&T` | `*const T` |
|--|------|------------|
| Safe? | Yes | No |
| Null ho sakta? | Never | Yes |
| Compiler check | Full | None |
| Dereference | Anywhere | unsafe only |
| Use case | Normal code | FFI, low-level |

## Q9. Kab use karte hain?

1. C library FFI: `extern "C" { fn strlen(s: *const i8) -> usize; }`
2. Custom data structures: `struct Node { val: i32, next: *mut Node }`
3. Performance-critical low-level code

---

# 3. `Box<T>` — Heap Pointer

## Q1. Box kya hota hai?
Value ko heap pe store karta hai.

```rust
fn main() {
    let x = 5;            // stack pe
    let y = Box::new(5);  // heap pe
    println!("{}", *y);   // 5
}

// Stack: y(ptr) ----> Heap: [5]
```

## Q2. 3 main use cases?

### 1. Large Data
```rust
let big = Box::new([0i32; 1_000_000]);  // heap pe safe
```

### 2. Recursive Types
```rust
// ERROR — compiler ko size pata nahi
enum List { Cons(i32, List), Nil }

// FIX — pointer ka size fixed (8 bytes)
enum List { Cons(i32, Box<List>), Nil }
```

### 3. Trait Objects
```rust
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog),
    Box::new(Cat),
];
for a in &animals { a.sound(); }
```

## Q3. Ownership?

```rust
let a = Box::new(10);
let b = a;             // ownership move!
// println!("{}", a);  // ERROR
println!("{}", b);     // 10
```

## Q4. Auto memory free?

```rust
{
    let x = Box::new(100);
}  // x drop, heap automatically free!
```

## Q5. Box ka size?

```rust
std::mem::size_of::<Box<i32>>()          // 8 bytes (pointer!)
std::mem::size_of::<Box<[i32; 1_000_000]>>()  // 8 bytes! (still pointer)
```

## Q6. Box vs Reference (Corrected Table)

| | `&T` | `Box<T>` |
|--|------|----------|
| Pointer kahan? | Stack | Stack |
| Data kahan? | Jahan original hai | Hamesha Heap |
| Ownership | Borrow | Owned |
| Null? | Never | Never |
| Auto free? | N/A | Yes |

---

# 4. `Rc<T>` — Reference Counted Pointer

## Q1. Rc kya hota hai?
Ek value ke multiple owners ho sakte hain!

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(10);
    let b = Rc::clone(&a);  // data copy nahi, sirf counter++
    let c = Rc::clone(&a);

    println!("{} {} {}", a, b, c);  // 10 10 10
    println!("Count: {}", Rc::strong_count(&a));  // 3
}
```

## Q2. Internally kaise kaam karta hai?

```rust
let a = Rc::new(10);   // count: 1
let b = Rc::clone(&a); // count: 2
drop(b);               // count: 1
// a drop -> count: 0 -> memory free!
```

Heap mein: { value: 10, strong_count: 3 }

## Q3. Rc::clone vs normal clone?

```rust
let b = Rc::clone(&a);  // cheap — sirf counter++
let c = (*a).clone();   // expensive — data copy!
```

## Q4. Mutation?

```rust
// *a = 20;  ERROR — Rc immutable hai by default
// Fix: Rc<RefCell<T>> use karo
```

## Q5. Thread safety?

```rust
// ERROR — Rc cannot be sent between threads!
// Fix: Arc<T> use karo
```

## Q6. Circular Reference = Memory Leak!

```
a ----> b
^       |
|_______|
Count hamesha >= 1, memory kabhi free nahi!
Fix: Weak<T> use karo
```

---

# 5. `Arc<T>` — Atomic Reference Counted

## Q1. Arc kya hota hai?
Rc ka thread-safe version!

```rust
use std::sync::Arc;

fn main() {
    let a = Arc::new(10);
    let b = Arc::clone(&a);
    println!("Count: {}", Arc::strong_count(&a));  // 2
}
```

## Q2. Rc vs Arc fark?

```
Rc<T>  -> Normal counter++ (fast, single thread only)
Arc<T> -> Atomic counter++ (thread safe, slight overhead)
```

Rule: Single threaded -> Rc (faster). Multi-threaded -> Arc.

## Q3. Arc with threads?

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);
    let mut handles = vec![];

    for i in 0..3 {
        let d = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            println!("Thread {}: {:?}", i, d);
        }));
    }

    for h in handles { h.join().unwrap(); }
}
```

## Q4. Arc se mutate karna? -> Arc<Mutex<T>>

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..5 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            *c.lock().unwrap() += 1;
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("Final: {}", *counter.lock().unwrap());  // 5
}
```

Arc<Mutex<T>> = Rust ka most common thread-safe pattern!

## Full Comparison Table

| | `Box<T>` | `Rc<T>` | `Arc<T>` |
|--|----------|---------|---------|
| Owners | 1 | Many | Many |
| Thread safe | Yes | No | Yes |
| Mutable | Yes | No | No (need Mutex) |
| Overhead | None | Counter | Atomic Counter |
| Use case | Single owner heap | Multi-owner 1 thread | Multi-owner multi-thread |

## Decision Tree

```
Value heap pe chahiye?
|-- NO  -> &T / &mut T
|-- YES -> Single owner?
           |-- YES -> Box<T>
           |-- NO  -> Single thread?
                      |-- YES -> Rc<T>
                      |          |-- Mutate bhi? -> Rc<RefCell<T>>
                      |-- NO  -> Arc<T>
                                 |-- Mutate bhi? -> Arc<Mutex<T>>
```

---

# Coming Soon

| # | Topic |
|---|-------|
| 6 | `Weak<T>` — Circular reference fix |
| 7 | `Cell<T>` / `RefCell<T>` — Interior Mutability |
