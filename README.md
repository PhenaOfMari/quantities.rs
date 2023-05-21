Unit-safe computations with quantities.

# What is a Quantity?

"The value of a quantity is generally expressed as the product of a number
and a unit. The unit is simply a particular example of the quantity concerned
which is used as a reference, and the number is the ratio of the value of the
quantity to the unit." (Bureau International des Poids et Mesures: The
International System of Units, 8th edition, 2006)

**Basic** types of quantities are defined "by convention", they do not depend
on other types of quantities, for example Length, Mass or Duration.

**Derived** types of quantities, on the opposite, are defined as products of
other types of quantities raised by some exponent.

Examples:

* Volume = Length ³

* Speed = Length ¹ · Duration ⁻¹

* Acceleration = Length ¹ · Duration ⁻²

* Force = Mass ¹ · Acceleration ¹

Each type of quantity may have one special unit which is used as a reference
for the definition of all other units, for example Meter, Kilogram and
Second. The other units are then defined by their relation to the reference
unit.

If a type of quantity is derived from types of quantities that all have a
reference unit, then the reference unit of that type is defined by a formula
that follows the formula defining the type of quantity.

Examples:

* Speed -> Meter per Second = Meter ¹ · Second ⁻¹

* Acceleration -> Meter per Second squared = Meter ¹ · Second ⁻²

* Force -> Newton = Kilogram ¹ · Meter ¹ · Second ⁻²


# "Systems of Measure"

There may be different systems which define quantities, their units and the
relations between these units in a different way.

This is not directly supported by this package. For each type of quantity
there can be only no or exactly one reference unit. But, if you have units
from different systems for the same type of quantity, you can define these
units and provide mechanisms to convert between them.

# The Basics: Quantity and Unit

The essential functionality of the package is provided by the traits 
`Quantity`, `HasRefUnit`, `Unit` and `LinearScaledUnit` and a macro generating
the code for concrete types of quantities.

A **basic** type of quantity can easily be defined using the proc-macro
attribute `quantity`, optionally followed by an attribute `refunit` and
followed by at least one attribute `unit`.

The macro generates an enum with the given units (incl. the refunit, if given)
as variants (together with an implemention of trait `Unit` and, if there's a
reference unit, of trait `LinearScaledUnit`), a type named after the given
struct, an implementation of trait `Quantity` for this type, an implementation
of trait `HasRefUnit` in case there's a reference unit, as well as
implementations of some std traits.

In addition, it creates a constant for each enum variant, thus providing a
constant for each unit. This implies that the identifiers of all units over
all defined quantitities have to be unique!

Example:

```rust
# use quantities::prelude::*;
#[quantity]
#[ref_unit(Kilogram, "kg", KILO)]
#[unit(Milligram, "mg", MILLI, 0.000001)]
#[unit(Carat, "ct", 0.0002)]
#[unit(Gram, "g", NONE, 0.001)]
#[unit(Ounce, "oz", 0.028349523125)]
#[unit(Pound, "lb", 0.45359237)]
#[unit(Stone, "st", 6.35029318)]
#[unit(Tonne, "t", MEGA, 1000.)]
/// The quantity of matter in a physical body.
struct Mass {}

assert_eq!(MILLIGRAM.name(), "Milligram");
assert_eq!(POUND.symbol(), "lb");
assert_eq!(TONNE.si_prefix(), Some(SIPrefix::MEGA));
assert_eq!(CARAT.scale(), Amnt!(0.0002));
```

In order to create a **derived** type of quantity based on more basic types of
quantities, an expression can be given as argument to the proc-macro attribute
`quantity`, specifying the quantity as product or as quotient of two base
quantities.

Instances of a derived quantity can then be build by multiplying or dividing
instances of the base quantities. Instances of a quantity which is defined
as a product can be divided by an instance of one of the base quantities,
giving an instance of the other base quantity as result. Instances of a
quantity which is defined as a quotient can be multiplied by an instance of
the divisor quantity, resulting in an instance of the divident quantity.

Example:

```rust
# use quantities::prelude::*;
#[quantity]
#[ref_unit(Meter, "m", NONE, "Reference unit of quantity `Length`")]
#[unit(Centimeter, "cm", CENTI, 0.01, "0.01·m")]
#[unit(Kilometer, "km", KILO, 1000, "1000·m")]
#[unit(Mile, "mi", 1609.344, "8·fur")]
pub struct Length {}

#[quantity]
#[ref_unit(Second, "s", NONE, "Reference unit of quantity `Duration`")]
#[unit(Minute, "min", 60, "60·s")]
#[unit(Hour, "h", 3600, "60·min")]
pub struct Duration {}

#[quantity(Length * Length)]
#[ref_unit(Square_Meter, "m²", NONE, "Reference unit of quantity `Area`")]
#[unit(Square_Centimeter, "cm²", 0.00001, "cm²")]
#[unit(Square_Kilometer, "km²", MEGA, 1000000., "km²")]
pub struct Area {}

let a = Amnt!(3.) * METER;
let b = Amnt!(0.5) * KILOMETER;
let ab = a * b;
assert_eq!(ab, Amnt!(1500.) * SQUARE_METER);
let c = ab / (Amnt!(2.) * KILOMETER);
assert_eq!(c, Amnt!(0.75) * METER);

#[quantity(Length / Duration)]
#[ref_unit(Meter_per_Second, "m/s", NONE, "Reference unit of quantity `Speed`")]
#[unit(Miles_per_Hour, "mph", 0.44704, "mi/h")]
pub struct Speed {}

let l = Amnt!(150.) * MILE;
let t = Amnt!(1.2) * HOUR;
let v = l / t;
assert_eq!(v, Amnt!(125.) * MILES_PER_HOUR);
let d = v * Duration::new(Amnt!(3.), HOUR);
assert_eq!(d, Amnt!(375.) * MILE);
```

# Type of the numerical part

The package allows to use either `float` or `fixed-point decimal` values for
the numerical part of a `Quantity` value.

Internally the type alias `AmountT` is used. This alias can be controlled by
the optional feature `fpdec` (see [features](#crate-features)).

When feature `fpdec` is off (= default), `AmountT` is defined as `f64` on a
64-bit system or as `f32` on a 32-bit system.

When feature `fpdec` is activated, `AmountT` is defined as `Decimal` (imported
from crate `fpdec`).

The macro `Amnt!` can be used to convert float literals correctly to `AmountT`
depending on the configuration. This is done automatically for the scale 
values of units by the proc-macro `quantity` described above.

# Instantiating quantities

An instance of a quantity type can be created by calling the function `new`,
giving an amount and a unit. Alternatively, an amount and a unit can be
multiplied.

Example:

```rust
# use quantities::prelude::*;
# #[quantity]
# #[ref_unit(Kilogram, "kg", KILO)]
# #[unit(Gram, "g", NONE, 0.001)]
# struct Mass {}
let m = Mass::new(Amnt!(17.4), GRAM);
assert_eq!(m.to_string(), "17.4 g");
let m = Amnt!(17.4) * GRAM;
assert_eq!(m.to_string(), "17.4 g");
```

# Unit-safe computations

If the quantity type has a refernce unit, a quantity instance can be converted
to a quantity instance with a different unit of the same type by calling the
method `convert`.

Example:

```rust
# use quantities::prelude::*;
# #[quantity]
# #[ref_unit(Kilogram, "kg", KILO)]
# #[unit(Carat, "ct", 0.0002)]
# #[unit(Gram, "g", NONE, 0.001)]
# struct Mass {}
let x = Mass::new(Amnt!(13.5), GRAM);
let y = x.convert(CARAT);
assert_eq!(y.to_string(), "67.5 ct");
```

Quantity values with the same unit can always be added or subtracted. Adding
or subtracting values with different units requires the values to be
convertable.

Example:

```rust
# use quantities::prelude::*;
# #[quantity]
# #[ref_unit(Kilogram, "kg", KILO)]
# #[unit(Gram, "g", NONE, 0.001)]
# struct Mass {}
let x = Amnt!(17.4) * GRAM;
let y = Amnt!(1.407) * KILOGRAM;
let z = x + y;
assert_eq!(z.amount(), Amnt!(1424.4));
assert_eq!(z.unit(), GRAM);
let z = y + x;
assert_eq!(z.to_string(), "1.4244 kg");
```

Quantity values can always be multiplied or divided by numerical values, 
preserving the unit.

Example:

```rust
# use quantities::prelude::*;
# #[quantity]
# #[ref_unit(Kilogram, "kg", KILO)]
# #[unit(Gram, "g", NONE, 0.001)]
# struct Mass {}
let x = Amnt!(7.4);
let y = Amnt!(1.7) * KILOGRAM;
let z = x * y;
assert_eq!(z.to_string(), "12.58 kg");
```

# Commonly Used Quantities

The package provides optional modules with definitions of commonly used
quantities; each can be activated by a feature with a corresponding name
(see [below](#predefined-quantities)).

# Crate features

By default, only the feature `std` is enabled.

## Ecosystem

* **std** - When enabled, this will cause `quantities` to use the standard
  library, so that conversion to string, formatting and printing are 
  available. When disabled, the use of crate `alloc` together with a
  system-specific allocator is needed to use that functionality.

## Optional dependencies

* **fpdec** - When enabled, instead of `f64` or `f32` `fpdec::Decimal` is used
  as `AmountT` (see [above](#type-of-the-numerical-part)).

* **serde** - When enabled, support for `serde` is enabled.

## Predefined quantities

With the following features additional modules can be enabled, each providing
a predefined quantity.

* **mass** - module [mass] - quantity [Mass](mass::Mass)
* **length** - module [length] - quantity [Length](length::Length)
* **duration** - module [duration] - quantity [Duration](duration::Duration)
* **area** - module [area] - quantity [Area](area::Area)
* **volume** - module [volume] - quantity [Volume](volume::Volume)
* **speed** - module [speed] - quantity [Speed](speed::Speed)
* **acceleration** - module [acceleration] - quantity 
  [Acceleration](acceleration::Acceleration)
* **force** - module [force] - quantity [Force](force::Force)
* **energy** - module [energy] - quantity [Energy](energy::Energy)
* **power** - module [power] - quantity [Power](power::Power)
* **frequency** - module [frequency] - quantity 
  [Frequency](frequency::Frequency)
* **datavolume** - module [datavolume] - quantity
  [DataVolume](datavolume::DataVolume)
* **datathroughput** - module [datathroughput] - quantity
  [DataThroughput](datathroughput::DataThroughput)
* **temperature** - module [temperature] - quantity
  [Temperature](temperature::Temperature)
