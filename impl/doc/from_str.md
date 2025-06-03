# What `#[derive(FromStr)]` generates

Deriving `FromStr` only works for enums/structs with no fields
or newtypes (structs or enum variants with only a single field).
The result is that you will be able to call the `parse()` method
on a string to convert it to your newtype. This only works when
the wrapped type implements `FromStr` itself.




## Structs

Structs deriving `FromStr` may either be empty or have a single field.

If the struct is empty, then the input string matches when it matches
the name of the struct in any case, unless `rename_all = "..."` is used
to force matching a specific case. Alternatively, `rename = "..."` can be used
to change the string that matches instead of the name of the struct.
`alias = "..."` can be used to add additional strings that match.

If the struct has a single field, then the input string is parsed as if it were
the type of the field.

When deriving `FromStr` for a struct:
```rust
# use derive_more::FromStr;
#[derive(FromStr, Debug, Eq, PartialEq)]
struct MyEmptyStruct;

// by default, strings matching the name of the struct are matched
// by lowercasing the input
assert_eq!("myemptystruct".parse::<MyEmptyStruct>().unwrap(), MyEmptyStruct);
assert_eq!("MyEmptyStruct".parse::<MyEmptyStruct>().unwrap(), MyEmptyStruct);

#[derive(FromStr, Debug, Eq, PartialEq)]
// use `rename = "..."` to change the string that matches instead of
// the name of the struct
#[from_str(rename = "my-cool-struct")]
// use `alias = "..."` to add additional strings that match
#[from_str(alias = "my-uncool-struct")]
struct MyEmptyStruct2;

assert_eq!("my-cool-struct".parse::<MyEmptyStruct2>().unwrap(), MyEmptyStruct2);
assert_eq!("my-uncool-struct".parse::<MyEmptyStruct2>().unwrap(), MyEmptyStruct2);
assert!("MyEmptyStruct2".parse::<MyEmptyStruct2>().is_err())

#[derive(FromStr, Debug, Eq, PartialEq)]
// use `rename_all = "..."` to force matching the name of the
// struct but using the given case
#[from_str(rename_all = "SCREAMING_SNAKE_CASE")]
struct MyEmptyStruct3;

assert_eq!("MY_EMPTY_STRUCT3".parse::<MyEmptyStruct3>().unwrap(), MyEmptyStruct3);
assert!("my_empty_struct3".parse::<MyEmptyStruct3>().is_err());

#[derive(FromStr, Debug, Eq, PartialEq)]
// newtype structs forward the `FromStr` implementation to the inner type
struct MyInt(i32);

assert_eq!("123".parse::<MyInt>().unwrap(), MyInt(123));
assert_eq!("0x123".parse::<MyInt>().unwrap(), MyInt(0x123));
assert_eq!("0b1111001".parse::<MyInt>().unwrap(), MyInt(123));
assert!("abc".parse::<MyInt>().is_err());

#[derive(FromStr, Debug, Eq, PartialEq)]
struct Point1D {
    x: i32,
}

assert_eq!("100".parse::<Point1D>().unwrap(), Point1D { x: 100 });
```

Code like this is generated:
```rust
# struct MyEmptyStruct;
impl derive_more::core::str::FromStr for MyEmptyStruct {
    type Err = derive_more::FromStrError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        _ if s.eq_ignore_ascii_case("Foo") => Self,
        _ => {
            return derive_more::core::result::Result::Err(
                derive_more::FromStrError::new("Foo"),
            )
        }
    }
}

# struct MyEmptyStruct2;
impl derive_more::core::str::FromStr for MyEmptyStruct2 {
    type Err = derive_more::FromStrError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        "my-cool-struct" => Self,
        "my-uncool-struct" => Self,
        _ => {
            return derive_more::core::result::Result::Err(
                derive_more::FromStrError::new("Foo"),
            )
        }
    }
}

# struct MyEmptyStruct3;
impl derive_more::core::str::FromStr for MyEmptyStruct3 {
    type Err = derive_more::FromStrError;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        "MY_EMPTY_STRUCT3" => Self,
        _ => {
            return derive_more::core::result::Result::Err(
                derive_more::FromStrError::new("Foo"),
            )
        }
    }
}

# struct MyInt(i32);
impl derive_more::core::str::FromStr for MyInt {
    type Err = <i32 as derive_more::core::str::FromStr>::Err;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        Ok(Self(i32::from_str(s)?))
    }
}

# struct Point1D {
#     x: i32,
# }
impl derive_more::core::str::FromStr for Point1D {
    type Err = <i32 as derive_more::core::str::FromStr>::Err;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        Ok(Self {
            x: i32::from_str(s)?,
        })
    }
}
```




## Enums

Enums deriving `FromStr` may have variants that are either empty or have
a single field.

By default, the input string matches the variant when it matches the
name of the variant in any case. If `rename_all = "..."` is used (on either
the enum or the variant), input string only matches the variant's name
converted to the specified case. Alternatively, `rename = "..."` can be used
to specify an exact match string for the variant. `alias = "..."` can be used
to specify additional match strings for the variant. `skip` can be used to
specify that this variant is never matched. `default` can be used to specify
that this variant is matched when no other variant matches, and the enum's
`FromStr::Err` associated type becomes `std::convert::Infallible`.

If the variant has a single field, then `transparent` must be specified,
which causes the `FromStr` implementation of the variant's field type to
match the input string. These variants are only matched when the field
type's `FromStr` implementation returns `Ok`.

When deriving `FromStr` for an enum:
```rust
# use derive_more::FromStr;
#[derive(FromStr, Debug, Eq, PartialEq)]
enum EnumNoFields {
    Foo,
    Bar,
    Baz,
    BaZ,
}

assert_eq!("foo".parse::<EnumNoFields>().unwrap(), EnumNoFields::Foo);
assert_eq!("Foo".parse::<EnumNoFields>().unwrap(), EnumNoFields::Foo);
assert_eq!("FOO".parse::<EnumNoFields>().unwrap(), EnumNoFields::Foo);

assert_eq!("Bar".parse::<EnumNoFields>().unwrap(), EnumNoFields::Bar);
assert_eq!("bar".parse::<EnumNoFields>().unwrap(), EnumNoFields::Bar);

assert_eq!("Baz".parse::<EnumNoFields>().unwrap(), EnumNoFields::Baz);
assert_eq!("BaZ".parse::<EnumNoFields>().unwrap(), EnumNoFields::BaZ);
assert_eq!(
    "other".parse::<EnumNoFields>().unwrap_err().to_string(),
    "Invalid `EnumNoFields` string representation",
);

#[derive(FromStr, Debug, Eq, PartialEq)]
#[from_str(rename_all = "SCREAMING-KEBAB-CASE")]
enum MyCoolEnum {
    #[from_str(rename_all = "lowercase", alias = "foo is cool")]
    Foo,
    Bar,
    #[from_str(skip)]
    BaR,
    #[from_str(default)]
    Baz,
    FooBar,
    #[from_str(forward)]
    Quux(i32),
}

assert_eq!("foo".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::Foo);
assert_eq!("foo is cool".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::Foo);
// since `rename_all = "lowercase"` is specified for `Foo`,
// only `lowercase` is matched and `SCREAMING-KEBAB-CASE` is ignored
assert_eq!("FOO".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::Baz);

// but `SCREAMING-KEBAB-CASE` is matched for other variants
assert_eq!("BAR".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::Bar);
assert_eq!("FOO-BAR".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::FooBar);

// numbers get successfully parsed by `i32::from_str`, resulting in Quux
assert_eq!("42".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::Quux(42));
assert_eq!("0x42".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::Quux(0x42));

// since `Baz` is marked `default`, any string not matched by other variants
// will result in `Baz` being returned
assert_eq!("FOOBAR".parse::<MyCoolEnum>().unwrap(), MyCoolEnum::Baz);
assert_eq!(
    "xXx_ this string can be anything _xXx".parse::<MyCoolEnum>().unwrap(),
    MyCoolEnum::Baz,
);
```

Code like this is generated:
```rust
# enum EnumNoFields {
#     Foo,
#     Bar,
#     Baz,
#     BaZ,
# }
impl derive_more::core::str::FromStr for EnumNoFields {
    type Err = derive_more::FromStrError;
    fn from_str(s: &str) -> Result<Self, derive_more::FromStrError> {
        Ok(match s {
            _ if s.eq_ignore_ascii_case("Foo") => Self::Foo,
            _ if s.eq_ignore_ascii_case("Bar") => Self::Bar,
            "Baz" => Self::Baz,
            "BaZ" => Self::BaZ,
            _ => return Err(derive_more::FromStrError::new("EnumNoFields")),
        })
    }
}

# enum MyCoolEnum {
#     Foo,
#     Bar,
#     BaR,
#     Baz,
#     FooBar,
#     Quux(i32),
# }
impl derive_more::core::str::FromStr for MyCoolEnum {
    type Err = core::convert::Infallible;
    fn from_str(s: &str) -> derive_more::core::result::Result<Self, > {
        if let Some(v) = derive_more::core::str::FromStr::from_str(s)
            .map(|v| Self::Quux(v))
            .ok()
        {
            return Ok(v);
        }
        derive_more::core::result::Result::Ok(match s {
            "foo is cool" => Self::Foo,
            "foo" => Self::Foo,
            "BAR" => Self::Bar,
            "FOO-BAR" => Self::FooBar,
            _ => Self::Baz,
        })
    }
}
```
