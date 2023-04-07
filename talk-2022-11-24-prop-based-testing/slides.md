---
theme: penguin
class: text-center
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
css: unocss
title: 'Property Based Testing'

themeConfig:
  twitter: '@greyblake'
  # blank image
  eventLogo: 'data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw=='
---

---
layout: intro
---

# Property-Based Testing


with Arbitrary and arbtest


by Serhii Potapov

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)fdsf


Всім доброго вечеора,
сьогодні я буду розповідати про property-based testing в Rust з
використанням бібліотека Arbitrary та arbtest.
Також ми розглянемо один практичний use-case, де property-based testing
може бути досить корисним.
-->





---
layout: presenter
presenterImage: 'https://www.greyblake.com/greyblake.jpeg'
---

# Serhii Potapov

**Full Stack Engineer at Impero.com**

<v-clicks>

* ⚓ Berlin/Kharkiv
* 🧔 Doing web development since 2008
* 🧑‍💻 Open Source (whatlang, nutype, ta, envconfig)
* 📝 Blog: <a href="https://greyblake.com">greyblake.com</a>
* <img src="/img/rustcast.png" width="32" style="display: inline-block"/> RustCast (youtube channel)
* 🌐 Github, Twitter, etc: <a href="https://twitter.com/greyblake">@greyblake</a>
* <img src="https://upload.wikimedia.org/wikipedia/commons/f/f5/Flag_of_Esperanto.svg" width="32" style="display: inline-block"/> I speak Esperanto

</v-clicks>

---


## Practical problem

<div class="grid grid-cols-2 gap-4">
<div>

Domain model
```rust
struct Vehicle {
    id: VehicleId,
    vehicle_type: VehicleType,
}

struct VehicleId(i32);

enum VehicleType {
    Car {
        fuel: Fuel,
        max_speed_kph: Option<u32>,
    },
    Bicycle,
}

enum Fuel {
    Electricity,
    Diesel,
}
```
</div>
<v-clicks>
<div>

Database record
```rust
struct VehicleRecord {
    id: i32,

    // Either "Car" or "Bicycle".
    kind: String,

    // Always None for Bicycle.
    // "Electricity" or "Diesel" for Car.
    fuel: Option<String>,

    // Always None for Bicycle. Optional for Car.
    // NOTE: PostgreSQL has no unsigned integers,
    // so we use i32 here.
    max_speed_kph: Option<i32>,
}
```

</div>
</v-clicks>
</div>


---

# Mappers

```rust {all|1}
fn vehicle_to_record(vehicle: Vehicle) -> VehicleRecord {
    let Vehicle { id, vehicle_type } = vehicle;
    let (kind, fuel, max_speed_kph) = match vehicle_type {
        VehicleType::Car { fuel, max_speed_kph } => (
            "Car".to_owned(),
             Some(fuel_to_str(fuel).to_owned()),
             max_speed_kph.map(|speed| speed.try_into().unwrap() )
        ),
        Bicycle => ("Bicycle".to_owned(), None, None,)
    };
    VehicleRecord { id: id.0, kind, fuel, max_speed_kph }
}

fn fuel_to_str(fuel: Fuel) -> &'static str {
    match fuel {
        Fuel::Electricity => "Electricity",
        Fuel::Diesel => "Diesel",
    }
}
```

---

```rust {all|1}
fn record_to_vehicle(record: VehicleRecord) -> Vehicle {
    let VehicleRecord { id, kind, fuel, max_speed_kph } = record;
    let id = VehicleId(id);
    let vehicle_type = match kind.as_str() {
        "Car" => {
            let fuel: String = fuel.expect("Car must have fuel");
            VehicleType::Car {
                fuel: fuel_from_str(&fuel),
                max_speed_kph: max_speed_kph.map(|speed| speed.try_into().unwrap() ),
            }
        },
        "Bicycle" => VehicleType::Bicycle,
        unknown => panic!("Unknown vehicle type: {unknown}")
    };
    Vehicle { id, vehicle_type }
}

fn fuel_from_str(s: &str) -> Fuel {
    match s {
        "Electricity" => Fuel::Electricity,
        "Diesel" => Fuel::Diesel,
        fuel => panic!("Unknown fuel: {fuel}")
    }
}
```





---

<div class="grid grid-cols-2 gap-4">
<div>

## Unit test

```rust{all|3-9|10|11|12|all}
#[test]
fn test_vehicle_record_mapping() {
    let vehicle = Vehicle {
        id: VehicleId(123),
        vehicle_type: VehicleType::Car {
            fuel: Fuel::Electricity,
            max_speed_kph: None,
        }
    };
    let record = vehicle_to_record(vehicle.clone());
    let same_vehicle = record_to_vehicle(record);
    assert_eq!(vehicle, same_vehicle);
}
```

</div>
<div>
</div>
</div>

---

<div class="grid grid-cols-2 gap-4">
<div>

## Unit test

```rust
#[test]
fn test_vehicle_record_mapping() {
    let vehicle = Vehicle {
        id: VehicleId(123),
        vehicle_type: VehicleType::Car {
            fuel: Fuel::Electricity,
            max_speed_kph: None,
        }
    };
    let record = vehicle_to_record(vehicle.clone());
    let same_vehicle = record_to_vehicle(record);
    assert_eq!(vehicle, same_vehicle);
}
```
</div>
<div>




## Потенційні проблеми

<v-clicks>

* The test is biased
* Не покриває багато інших можливих випадків, наприклад:
  * `fuel` is `Diesel`
  * `max_speed_kph` is present
  * `VehicleType` is `Bicycle`
* Щоб покрити всі випадки необхідно багато схожих тестів


</v-clicks>

</div>
</div>

<!--
Треба мати на увазі, що складність тестів буде рости експоненціально з більшими структурами.
-->


----

```rust{all|1-20|21-25}
let vehicles = [
    Vehicle {
        id: VehicleId(123),
        vehicle_type: VehicleType::Car {
            fuel: Fuel::Electricity,
            max_speed_kph: None,
        }
    },
    Vehicle {
        id: VehicleId(124),
        vehicle_type: VehicleType::Car {
            fuel: Fuel::Diesel,
            max_speed_kph: Some(260),
        }
    },
    Vehicle {
        id: VehicleId(125),
        vehicle_type: VehicleType::Bicycle,
    },
]
for vehicle in vehicles.into_iter() {
    let record = vehicle_to_record(vehicle.clone());
    let same_vehicle = record_to_vehicle(record);
    assert_eq!(vehicle, same_vehicle);
}
```



---
layout: new-section
---

# Property-based testing to rescue!

<center>
<img src="img/Minion02.gif" width="500" />
</center>



## Arbitrary & arbtest

---

## Що таке **property-based testing**?

<v-clicks>

**Property-based tests** перевіряють, що предмет тестування має певну властивість (property).



Наприклад:

</v-clicks>


---

## Що таке **property-based testing**?


**Property-based tests** перевіряють, що предмет тестування має певну властивість (property).


Наприклад:


* `vehicle_to_record(vehicle: Vehicle) -> VehicleRecord` never panics

<v-clicks>

* `record_to_vehicle(vehicle_to_record(vehicle)) = vehicle`

</v-clicks>

---


## Як працює **property-based testing**?

<v-clicks>

* Generate a random input
* Feed it to the software (function, whatever)
* Verify software behaviour is correct
* Repeat until a failure is detected or time is up

</v-clicks>


---

<div class="grid grid-cols-2 gap-4">
<div>

### The old test

```rust
#[test]
fn test_vehicle_record_mapping() {
    let vehicles = [
        Vehicle {
            id: VehicleId(123),
            vehicle_type: VehicleType::Car {
                fuel: Fuel::Electricity,
                max_speed_kph: None,
            }
        },
        // full definition is hidden to save
        // some space on the slide
        Vehicle { ... },
        Vehicle { ... },
    ]
    for vehicle in vehicles.into_iter() {
        let record = vehicle_to_record(vehicle.clone());
        let same_vehicle = record_to_vehicle(record);
        assert_eq!(vehicle, same_vehicle);
    }
}
```
</div>
</div>

---

<div class="grid grid-cols-2 gap-4">

<div>

### The old test

```rust{all|17-19}
#[test]
fn test_vehicle_record_mapping() {
    let vehicles = [
        Vehicle {
            id: VehicleId(123),
            vehicle_type: VehicleType::Car {
                fuel: Fuel::Electricity,
                max_speed_kph: None,
            }
        },
        // full definition is hidden to save
        // some space on the slide
        Vehicle { ... },
        Vehicle { ... },
    ]
    for vehicle in vehicles.into_iter() {
        let record = vehicle_to_record(vehicle.clone());
        let same_vehicle = record_to_vehicle(record);
        assert_eq!(vehicle, same_vehicle);
    }
}
```
</div>

<div>

### Prop-based test

```rust {all|5-7|3,9|4}
#[test]
fn test_vehicle_record_mapping() {
    arbtest::builder().run(|u| {
        let vehicle = Vehicle::arbitrary(u)?;
        let record = vehicle_to_record(vehicle.clone());
        let same_vehicle = record_to_vehicle(record);
        assert_eq!(vehicle, same_vehicle);
        Ok(())
    })
}
```

</div>
</div>



---

## The test results

```rust {all|10|5}
failures:

---- test_vehicle_record_mapping stdout ----
thread 'test_vehicle_record_mapping' panicked at 'called `Result::unwrap()`
    on an `Err` value: TryFromIntError(())', src/main.rs:49:60
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


arb_test failed!
    Seed: 0x25dc50a20000003e
```


<v-clicks>

```rust {6}
// fn vehicle_to_record
    VehicleType::Car { fuel, max_speed_kph } => {
        (
            "Car".to_owned(),
            Some(fuel_to_str(fuel).to_owned()),
            max_speed_kph.map(|speed| speed.try_into().unwrap() )
        )
    }
```

</v-clicks>

---


## Reproduce the failure

```rust {all|2|4}
fn test_vehicle_record_mapping() {
    arbtest::builder().seed(0x25dc50a20000003e).run(|u| {
        let vehicle = Vehicle::arbitrary(u)?;
        dbg!(&vehicle);
        let record = vehicle_to_record(vehicle.clone());
        let same_vehicle = record_to_vehicle(record);
        assert_eq!(vehicle, same_vehicle);
        Ok(())
    })
}
```

<v-clicks>
Output:
```rust {all|5}
&vehicle = Vehicle {
    id: VehicleId(1455468422),
    vehicle_type: Car {
        fuel: Diesel,
        max_speed_kph: Some(2207965846),  // fits in u32, but too big for i32
    },
}
```
</v-clicks>


---

## Fix


```rust{1,5}
struct VehicleRecord {
    id: i32,
    kind: String,
    fuel: Option<String>,
    max_speed_kph: Option<i32>,
}
```

<div class="grid grid-cols-2 gap-4">
<v-clicks>
<div>
Before:
```rust{2,5}
#[derive(Debug, PartialEq, Clone, Arbitrary)]
enum VehicleType {
    Car {
        fuel: Fuel,
        max_speed_kph: Option<u32>,
    },
    Bicycle,
}
```
</div>
<div>
After:
```rust{2,5}
#[derive(Debug, PartialEq, Clone, Arbitrary)]
enum VehicleType {
    Car {
        fuel: Fuel,
        max_speed_kph: Option<u16>,
    },
    Bicycle,
}
```
</div>
</v-clicks>
</div>

Any `u16` f



---
layout: new-section
---

# Як працює Arbitrary та arbtest?

---

### Arbitrary trait

```rust {all|2}
pub trait Arbitrary<'a>: Sized {
    fn arbitrary(u: &mut Unstructured<'a>) -> Result<Self>;

    fn arbitrary_take_rest(u: Unstructured<'a>) -> Result<Self> { ... }
    fn size_hint(depth: usize) -> (usize, Option<usize>) { ... }
}
```

<br />

<v-clicks>

<div>

## Unstructured

```rust
pub struct Unstructured<'a> {
    data: &'a [u8],
}
```

</div>

</v-clicks>

---

## Derive

```rust {all|1,3,9,12,18}
use arbitrary::Arbitrary;

#[derive(Arbitrary)]
struct Vehicle {
    id: VehicleId,
    vehicle_type: VehicleType,
}

#[derive(Arbitrary)]
struct VehicleId(i32);

#[derive(Arbitrary)]
enum Fuel {
    Electricity,
    Diesel,
}

#[derive(Arbitrary)]
enum VehicleType {
    ...
}
```

---

<div class="grid grid-cols-2 gap-4">
<div>
Arbitrary vehicle

```rust {all|3|4|5}
fn main() {
    // Some random bytes
    let bytes: [u8; 16] = [255, 40, 179, 24, 184, ...];
    let mut u = arbitrary::Unstructured::new(&bytes);
    let vehicle = Vehicle::arbitrary(&mut u).unwrap();
    dbg!(&vehicle);
}
```
</div>

<div>

</div>
</div>



---


<div class="grid grid-cols-2 gap-4">
<div>
Arbitrary vehicle

```rust {6}
fn main() {
    // Some random bytes
    let bytes: [u8; 16] = [255, 40, 179, 24, 184, ...];
    let mut u = arbitrary::Unstructured::new(&bytes);
    let vehicle = Vehicle::arbitrary(&mut u).unwrap();
    dbg!(&vehicle);
}
```
</div>

<div>
Output

```
&vehicle = Vehicle {
    id: VehicleId(
        414394623,
    ),
    vehicle_type: Car {
        fuel: Electricity,
        max_speed_kph: Some(
            3870351,
        ),
    },
}
```

</div>
</div>






---


## Існуючий інструментарій

<v-clicks>

* Quickcheck
* Proptest
* Arbitrary & arbtest

</v-clicks>


---

## Порівняння

|                        | **Quickcheck** | **Proptest** | **Arbitrary** |
|------------------------|----------------|--------------|---------------|
| **prop-based testing** | ✔️              | ✔️            | ✔️             |

---

## Порівняння

|                        | **Quickcheck** | **Proptest** | **Arbitrary** |
|------------------------|----------------|--------------|---------------|
| **prop-based testing** | ✔️              | ✔️            | ✔️             |
| **fuzzy testing**      |                |              | ✔️             |

---

## Порівняння

|                        | **Quickcheck** | **Proptest** | **Arbitrary** |
|------------------------|----------------|--------------|---------------|
| **prop-based testing** | ✔️              | ✔️            | ✔️             |
| **fuzzy testing**      |                |              | ✔️             |
| **derive macro**       |                | ✔️            | ✔️             |


---

## Порівняння

|                        | **Quickcheck** | **Proptest** | **Arbitrary** |
|------------------------|----------------|--------------|---------------|
| **prop-based testing** | ✔️              | ✔️            | ✔️             |
| **fuzzy testing**      |                |              | ✔️             |
| **derive macro**       |                | ✔️            | ✔️             |
| **shrinking**          | ✔️              | ✔️            |               |


---

## Порівняння

|                                   | **Quickcheck** | **Proptest** | **Arbitrary** |
|-----------------------------------|----------------|--------------|---------------|
| **prop-based testing**            | ✔️              | ✔️            | ✔️             |
| **fuzzy testing**                 |                |              | ✔️             |
| **derive macro**                  |                | ✔️            | ✔️             |
| **shrinking**                     | ✔️              | ✔️            |               |
| **integration with other crates** |                |              | ✔️             |


---

## Порівняння

|                                   | **Quickcheck** | **Proptest** | **Arbitrary** |
|-----------------------------------|----------------|--------------|---------------|
| **prop-based testing**            | ✔️              | ✔️            | ✔️             |
| **fuzzy testing**                 |                |              | ✔️             |
| **derive macro**                  |                | ✔️            | ✔️             |
| **shrinking**                     | ✔️              | ✔️            |               |
| **integration with other crates** |                |              | ✔️             |
| **multi strategies**              |                | ✔️            |               |


---


# Переваги


<v-clicks>

* Not biased test input
* Дуже добре підходить для тестування:
  * Небажаних panics (e.g. is often used for parsing libraries)
  * Cиметричного перетворення даних
* Не потребує багато зусиль

</v-clicks>



---

# Можливі недоліки:

<v-clicks>

* Не є 100% заміною юніт тестам
* Недетерменовані тести можуть інколи ломати CI (але це скоріше добре, бо знайдена помилка, про яку раніше не знали!).

</v-clicks>


---

<center>
    <h1> Дякую! </h1>
</center>
