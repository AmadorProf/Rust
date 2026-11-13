# Tema 6 — Traits y genéricos

## 6.1 Traits: interfaces con comportamiento compartido

Un **trait** define un conjunto de métodos que un tipo puede implementar. Es el equivalente a una *interface* de Java/Go, pero con la posibilidad de dar implementaciones por defecto (como las *default methods* de Java 8+). En C no existe nada parecido en el lenguaje: se simula con punteros a función dentro de structs.

```rust
trait Saludable {
    fn saludo(&self) -> String;

    // método con implementación por defecto: los tipos pueden heredarlo o sobreescribirlo
    fn saludo_formal(&self) -> String {
        format!("Buenos días. {}", self.saludo())
    }
}

struct Persona {
    nombre: String,
}

impl Saludable for Persona {
    fn saludo(&self) -> String {
        format!("Hola, soy {}", self.nombre)
    }
}

fn main() {
    let p = Persona { nombre: String::from("Ana") };
    println!("{}", p.saludo());
    println!("{}", p.saludo_formal()); // usa la implementación por defecto
}
```

## 6.2 Traits como parámetros: polimorfismo estático

```rust
fn presentar(entidad: &impl Saludable) {
    println!("{}", entidad.saludo());
}

// Azúcar sintáctico equivalente, forma explícita con genéricos (ver 6.3):
fn presentar2<T: Saludable>(entidad: &T) {
    println!("{}", entidad.saludo());
}
```

Esto es **despacho estático** (*static dispatch*): el compilador genera, para cada tipo concreto usado, una versión especializada de la función (monomorfización), igual que hacen los templates de C++. Cero overhead en tiempo de ejecución: no hay tabla de punteros a función, es código tan directo como si se hubiera escrito a mano para ese tipo específico.

## 6.3 Genéricos

```rust
fn mayor<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

fn main() {
    println!("{}", mayor(3, 7));       // funciona con i32
    println!("{}", mayor(3.5, 2.1));   // funciona con f64
    println!("{}", mayor("abc", "xyz")); // funciona con &str
}
```

`T: PartialOrd` es un **trait bound**: restringe `T` a tipos que implementan `PartialOrd` (es decir, que se pueden comparar con `<`, `>`). Sin este bound, el compilador rechazaría `a > b` porque no sabría si un `T` genérico cualquiera soporta esa operación. Esto es el equivalente, pero verificado en compilación y sin "duck typing" implícito, a lo que en C++ moderno serían los *concepts*.

Genéricos en structs:

```rust
struct Punto<T> {
    x: T,
    y: T,
}

impl<T: std::ops::Add<Output = T> + Copy> Punto<T> {
    fn suma_coordenadas(&self) -> T {
        self.x + self.y
    }
}

fn main() {
    let p_int = Punto { x: 3, y: 4 };
    let p_float = Punto { x: 1.5, y: 2.5 };
    println!("{}", p_int.suma_coordenadas());
    println!("{}", p_float.suma_coordenadas());
}
```

## 6.4 Trait objects: polimorfismo dinámico (cuando sí hace falta)

Cuando se necesita guardar en una misma colección valores de **tipos distintos** que comparten un trait (lo que en C se simularía con `void*` + una tabla de punteros a función manual, y en C++ con clases virtuales), se usa `dyn Trait`:

```rust
trait Figura {
    fn area(&self) -> f64;
}

struct Circulo { radio: f64 }
struct Cuadrado { lado: f64 }

impl Figura for Circulo {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radio * self.radio }
}
impl Figura for Cuadrado {
    fn area(&self) -> f64 { self.lado * self.lado }
}

fn main() {
    let figuras: Vec<Box<dyn Figura>> = vec![
        Box::new(Circulo { radio: 2.0 }),
        Box::new(Cuadrado { lado: 3.0 }),
    ];

    for f in &figuras {
        println!("Área: {:.2}", f.area());
    }
}
```

`Box<dyn Figura>` es un puntero al heap más una tabla de punteros a función (*vtable*), exactamente el mecanismo que usa C++ internamente para métodos virtuales — solo que en Rust es opt-in: se paga ese pequeño coste de indirección **solo** cuando se usa `dyn`, no por defecto como en los lenguajes orientados a objetos clásicos.

**Regla práctica**: usa genéricos (`impl Trait` / `<T: Trait>`) cuando el tipo concreto se conoce en compilación (más rápido, más optimizable); usa `dyn Trait` cuando necesitas mezclar tipos distintos en tiempo de ejecución (plugins, colecciones heterogéneas, callbacks configurables).

## 6.5 Traits importantes de la librería estándar

| Trait | Para qué sirve |
|---|---|
| `Debug` | Representación de depuración (`{:?}`) |
| `Display` | Representación para el usuario (`{}`) — hay que implementarlo a mano |
| `Clone` / `Copy` | Duplicar valores |
| `PartialEq` / `Eq` | Comparación `==` |
| `PartialOrd` / `Ord` | Comparación `<`, `>`, ordenar |
| `Default` | Valor por defecto, `T::default()` |
| `Iterator` | Convertir un tipo en algo iterable con `for` |
| `From` / `Into` | Conversiones entre tipos, idiomático en vez de constructores de conversión |

Ejemplo de `Display` + `From`:

```rust
use std::fmt;

struct Temperatura(f64); // grados Celsius

impl fmt::Display for Temperatura {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.1}°C", self.0)
    }
}

impl From<f64> for Temperatura {
    fn from(grados: f64) -> Self {
        Temperatura(grados)
    }
}

fn main() {
    let t: Temperatura = 36.6.into(); // Into es el reverso automático de From
    println!("{t}"); // usa Display: 36.6°C
}
```

## 6.6 Ejercicio

1. Define un trait `Resumible` con un método `fn resumen(&self) -> String`.
2. Impléntalo para dos structs distintos, p. ej. `Articulo` (título + cuerpo) y `Tweet` (usuario + texto), cada uno con su propia lógica de resumen.
3. Escribe una función `fn imprimir_resumen(item: &impl Resumible)` y pruébala con ambos tipos.
4. (Extra) Crea un `Vec<Box<dyn Resumible>>` con instancias de ambos tipos mezclados y recorre el vector imprimiendo todos los resúmenes.

**Siguiente →** [Tema 7: Manejo de errores](07-manejo-de-errores.md)
