# Rust traits workshop

Link to slides: <https://codeandbitters.com/slides/rust_traits.html>

---

# From trait

    pub trait From<T> {
        pub fn from(T) -> Self;
    }

---

# Into trait

    pub trait Into<T> {
        pub fn into(self) -> T;
    }

If `From<X> for Y`, exists, the std library implements `Into<Y> for X`.

---

# From and Into

preferred: implement `From`...

    struct MyType {
        inner: u64,
    }

    impl From<MyType> for u64 {
        pub fn from(x: MyType) -> Self {
            x.inner
        }
    }

... and accept `Into`:

    fn is_big<T>(val: T) -> bool
    where
        T: Into<u64>
    {
        let val: u64 = val.into();
        val > 1000000
    }

---

# TryFrom and TryInto

    
    pub trait TryFrom<T> {
        type Error;
        pub fn try_from(value: T) -> Result<Self, Self::Error>;
    }

    pub trait TryInto<T> {
        type Error;
        pub fn try_into(self) -> Result<T, Self::Error>;
    }

---

# Exercise

We want a struct that holds an integer between 0 and 100.

struct Percent(u8);

What traits should be implemented if we want to convert to/from a regular integer?

---

# AsRef trait

    pub trait AsRef<T> {
        pub fn as_ref(&self) -> &T;
    }

`AsRef` is for cheap reference to reference conversions.

There is also a trait called `AsMut` that does the same thing except with mutable references:

    pub trait AsMut<T> {
        pub fn as_mut(&mut self) -> &mut T;
    }

Note that `Option` and `Result` have member fns called `as_ref` and `as_mut` that do something different.

---

# without AsRef

    struct Foo {
        name: String
    }

    impl Foo {
        // only accepts an owned String
        fn new(name: String) -> Self {
            Self { name }
        }
    }

---

# without AsRef

    struct Foo {
        name: String
    }

    impl Foo {
        // only accepts a reference
        fn new(name: &str) -> Self {
            Self { name: String::from(name) }
        }
    }

---

# using AsRef

    struct Foo {
        name: String
    }

    impl Foo {
        // accepts String, &String, or &str
        fn new<S>(name: S) -> Self
        where
            S: AsRef<str>,
        {
            let name = String::from(name.as_ref());
            Self { name }
        }
    }

This works because `str` implements `AsRef<str>`, and so does `String`.

---

# Exercise

Write a function that creates a `PathBuf` of the form

    /tmp/{name}_{num}

where the name argument can be a `PathBuf`, `&Path`, `String`, or `&str` and the number argument is a u32.

Bonus: allow number to be u8, u16, or u32.

---

# Deref trait

    pub trait Deref {
        type Target;
        
        pub fn deref(&self) -> &Self::Target;
    }

`Deref` is intended for "smart pointers", types that own an inner type and want to inherit the inner type's capabilities.

The compiler will call `deref()` to reach the correct type. This is called "deref coercion".

    fn foo(s: &str) {}

    let s = String::from("hello world");
    foo(&s);

---

# Deref trait


`Deref` can be used along with autoref to transparently call methods
on the target type.

    fn main() {
        let s = String::from("hello world")
        println!("{}", s.starts_with("hello"));
    }

There is no `starts_with` member function for `String`, but one does exist for `str`.

---

# Deref trait

The compiler can call `deref` until it finds a type that works.

    fn main() {
        let s = Box::new(String::from("hello world));
        println!("{}", s.starts_with("hello"));
    }

autoref: `Box<String>` -> `&Box<String>`

deref coercion: `&Box<String>` -> `&String`

deref coercion: `&String` -> `&str`

---

# `AsRef` and `Deref`

`Deref` is intended for a type that contains only an inner type `T`, and wants to let you access all of T's methods.

The Deref docs say:
> Deref should only be implemented for smart pointers.

`AsRef` is for more general cases, e.g. types that contain an inner type `T`, but also some other data.

`AsRef` will not be used implicitly by the compiler.

Note that a type can `impl AsRef` for multiple types. `String` implements both `AsRef<[u8]>` and `AsRef<str>`

---

# Exercise

Create a wrapper type that stores an inner string.

Make it possible to call `String` methods with the dot-operator, e.g. `my_string.find("text")`

---

# FromStr

    pub trait FromStr {
        type Err;
        pub fn from_str(s: &str) -> Result<Self, Self::Err>;
    }

Any type that implements `FromStr` can be used with `str::parse`, for example:

    let addr: IpAddr = user_input.parse();

---

# Borrow and ToOwned

`Borrow` is very similar to `AsRef` but has additional rules:

- borrowed and owned form must compare equal (`Ord` and `Eq` traits)
- borrowed and owned form must have the same hash (`Hash` trait)

This trait is used in `Cow` and when specifying `HashMap` keys.

---

# Marker traits

A marker trait is just a trait that has no body.

    trait SomeMarker {}

There are some marker traits that are automatically generated by the compiler:
`Send`, `Sync`, `Sized`.

---

# Send and Sync

`Send`: this type may be moved to another thread.

`Sync`: this type may be accessed by multiple threads at once.

The compiler can determine this for most types.

Examples:

Raw pointers are not `Send` or `Sync`.

`Rc` is `Send` but not `Sync`.

---

# Sized

If a type has a fixed size in bytes, then the compiler will give it the `Sized` marker trait.

Things that are not sized:

`[T]` (e.g. [u8])

Trait objects (e.g. `dyn Read`)

    fn read_data(reader: &mut T)
    where
        T: Read + ?Sized
    {
        ...
    }

---

# Trait bounds

Functions, impl blocks, trait impls, etc. can specify a set of trait requirements.

Example:

    fn read_data<R>(reader: R) -> Result<()>
    where
        R: Read
    {
        let mut buffer = [0; 10];
        f.read_exact(&mut buffer)?;
    }

Alternate syntax, without "where":

    fn read_data<R: Read>(reader: R) -> Result<()>

---

# Trait bounds

Trait bounds can constrain associated types.

    fn stringify_first<It>(mut iterator: It) -> String
    where
        It: Iterator,
        It::Item: ToString,
    {
        ...
    }

---

# Trait bounds

There is a special trait bound 'static that is sometimes necessary:

    pub fn read_data_in_background<R>(reader: R)
    where
        R: Read + Send + 'static
    {
        thread::spawn(move || {
            let mut buf = Vec::<u8>::new();
            if let Ok(count) = f.read_to_end(&mut buf) {
                println!("read {} bytes from file.", count);
            }
        });
    }

---

# Trait bounds

Trait bounds can get pretty complicated.

    impl<K, V, S> HashMap<K, V, S>
    where
        K: Eq + Hash,
        S: BuildHasher, 
    {
        pub fn get<Q>(&self, k: &Q) -> Option<&V>
        where
            K: Borrow<Q>,
            Q: Hash + Eq + ?Sized
        {
            // ...
        }
    }

---

# Empty Traits

It's sometimes useful to declare a trait just for its bounds:

    pub trait NumOps<Rhs = Self, Output = Self>:
        Add<Rhs, Output = Output>
        + Sub<Rhs, Output = Output>
        + Mul<Rhs, Output = Output>
        + Div<Rhs, Output = Output>
        + Rem<Rhs, Output = Output>
    {
    }

(part of the `num` crate.)

---

# Blanket trait implementations

If we provide a trait implementation for a generic set of types, this is called a "blanket implementation".

Example from the std library:

    // From implies Into

    impl<T, U> Into<U> for T
    where
        U: From<T>,
    {
        fn into(self) -> U {
            U::from(self)
        }
    }

---

# Blanket trait implementations

[Example](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=32299872182a62dc1a4fec5de7df18dd):

    trait LoggedDrop {
        fn logged_drop(self);
    }

    impl<T> LoggedDrop for T
    where
        T: Debug,
    {
        fn logged_drop(self) {
            println!("dropping {:?}", self);
        }
    }

---

# Trait "extensions"

If you provide a blanket implementation, you can essentially "extend" an existing trait.

Example: tokio has `AsyncReadExt` which contains lots of useful things that you don't get with the plain `AsyncRead` trait.

---

# impl Trait

    fn foo(arg: impl Serialize) { ... }

...is basically the same as writing:

    fn foo<T>(arg: T)
    where
        T: Serialize
    { ... }

---

# impl Trait

"impl Trait" is particularly useful when returning a closure, which has a type that we cannot name.

    fn returns_closure() -> impl Fn(i32) -> i32 {
        |x| x + 1
    }

---

# Associated consts

    trait MessageId {
        const MSG_ID: u16;
    }

    impl MessageId for MyStruct {
        const MSG_ID: u16 = 1234;
    }

---

# Conflicting method names

    trait Foo {
        fn get(&self) -> &str;
    }

    trait Bar {
        fn get(&self) -> u32;
    }

    let val = SomeFooBarType::new();

    // Won't compile, ambiguous.
    let result = val.get();

    let foo_result = Foo::get(val);

or:

    let foo_result = <val as Foo>::get();

---

# impl Trait "orphan" rule

You can implement a foreign trait on a local type:

    struct Foo(u32);

    impl From<u32> for Foo {
        pub fn from(x: u32) -> Self {
            Self(x)
        }
    }

You can implement a local trait on a foreign type:

    trait Truthy {
        fn truthy(&self) -> bool;
    }

    impl Truthy for u32 {
        fn truthy(&self) -> bool {
            self != 0
        }
    }

---

# Foreign trait "orphan" rule

But you can't implement a foreign trait on a foreign type:

    impl Serialize for HashMap {
        // won't compile
    }

If you need to do something like this, the workaround is to create a wrapper type:

    struct MyMap(HashMap);

    impl Serialize for MyMap {
        ...
    }

---

# Foreign trait "orphan" rule

Rust 1.41 relaxed the foreign trait rule, to allow this:

    struct Foo;

    impl<T> From<Foo> for Vec<T> {
        fn from(_foo: Foo) -> Vec<T> {
            todo!()
        }
    }

Even though `From` and `Vec` are foreign, `From<Foo>` is a local type, so this impl is allowed.

---

# Object safety

To create a trait object for dynamic dispatch, trait methods must follow two rules:

- methods cannot return `Self`.
- methods cannot use generic type parameters.

Traits that violate these rules exist (e.g. `Clone`); you just can't create a trait object from them.

---

# Traits and async

Traits cannot currently contain `async fn`s.

There is a crate called `async-trait` that performs some macro magic to work around this restriction.

---

# Other RFCs / unstable features

[RFC 1598](https://rust-lang.github.io/rfcs/1598-generic_associated_types.html): Generic Associated Types (nightly feature `generic_associated_types`)

GATs may show up in stable Rust in late 2021.

[RFC 1733](https://rust-lang.github.io/rfcs/1733-trait-alias.html): Trait Aliases ([`trait_alias`](https://doc.rust-lang.org/beta/unstable-book/language-features/trait-alias.html))

---

# Other crates

[derive_more](https://docs.rs/derive_more/latest/derive_more/) and [educe](https://docs.rs/educe/latest/educe/) have derive macros for many std library traits.

[serde](https://docs.rs/serde/latest/serde/): Serialize, Deserialize, Serializer, Deserializer, DeserializeOwned

[num](https://docs.rs/num/latest/num/): Num, Integer, Float, Bounded, etc.

[futures](https://docs.rs/futures/latest/futures/): Stream, AsyncRead, AsyncWrite...
