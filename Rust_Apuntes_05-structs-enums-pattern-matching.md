# Tema 5 — Structs, enums y pattern matching

## 5.1 Structs: agrupar datos

Equivalente al `struct` de C, pero con métodos asociados (como una clase sin herencia).

```rust
struct Usuario {
    nombre: String,
    email: String,
    activo: bool,
}

fn main() {
    let user1 = Usuario {
        nombre: String::from("Ana"),
        email: String::from("ana@example.com"),
        activo: true,
    };

    println!("{} - {}", user1.nombre, user1.email);
}
```

### Métodos con `impl`

```rust
struct Rectangulo {
    ancho: f64,
    alto: f64,
}

impl Rectangulo {
    // "constructor" por convención, no es una palabra reservada
    fn nuevo(ancho: f64, alto: f64) -> Self {
        Rectangulo { ancho, alto } // field init shorthand: igual que { ancho: ancho, alto: alto }
    }

    // método que toma &self: préstamo inmutable del propio struct
    fn area(&self) -> f64 {
        self.ancho * self.alto
    }

    // método que necesita mutar el struct
    fn escalar(&mut self, factor: f64) {
        self.ancho *= factor;
        self.alto *= factor;
    }
}

fn main() {
    let mut r = Rectangulo::nuevo(3.0, 4.0);
    println!("Área: {}", r.area());
    r.escalar(2.0);
    println!("Área tras escalar: {}", r.area());
}
```

`self` es análogo a `this` en C++/Java, pero explícito y con las mismas reglas de ownership/borrowing que cualquier otra referencia: `&self` presta de forma inmutable, `&mut self` de forma mutable, y `self` (sin `&`) consume el struct (lo mueve), típico en builders o conversiones.

### Tuple structs y unit structs

```rust
struct Punto(f64, f64, f64); // útil cuando los nombres de los campos no aportan nada
let origen = Punto(0.0, 0.0, 0.0);

struct Marcador; // sin campos, útil como "etiqueta" para traits
```

### Derivar comportamiento común con `derive`

En C, para imprimir, comparar o copiar un struct hay que escribir el código a mano (o usar macros propias). En Rust, muchos comportamientos comunes se derivan automáticamente:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Punto2D {
    x: f64,
    y: f64,
}

fn main() {
    let p1 = Punto2D { x: 1.0, y: 2.0 };
    let p2 = p1.clone();
    println!("{p1:?}"); // Debug: imprime Punto2D { x: 1.0, y: 2.0 }
    println!("{}", p1 == p2); // PartialEq: true
}
```

- `Debug`: permite imprimir con `{:?}` (depuración). `Display` (no se deriva, hay que implementarlo) permite `{}` (salida para el usuario final).
- `Clone`: habilita `.clone()`.
- `PartialEq`/`Eq`: habilita `==`/`!=`.
- Otros comunes: `Default`, `Hash`, `PartialOrd`/`Ord`, `Copy` (solo para tipos sin recursos en heap).

## 5.2 Enums: mucho más potentes que en C

En C, un `enum` es básicamente una lista de enteros con nombre. En Rust, cada variante de un enum puede llevar **datos asociados de tipos distintos**, convirtiéndolo en lo que otros lenguajes llaman una "union etiquetada" (*tagged union* / *sum type*) de forma segura.

```rust
enum Mensaje {
    Salir,                       // sin datos
    Mover { x: i32, y: i32 },    // como un struct anónimo
    Escribir(String),            // un dato de tipo String
    CambiarColor(i32, i32, i32), // tupla de 3 valores
}

fn procesar(msg: Mensaje) {
    match msg {
        Mensaje::Salir => println!("Saliendo"),
        Mensaje::Mover { x, y } => println!("Mover a ({x}, {y})"),
        Mensaje::Escribir(texto) => println!("Escribir: {texto}"),
        Mensaje::CambiarColor(r, g, b) => println!("Color RGB({r},{g},{b})"),
    }
}

fn main() {
    procesar(Mensaje::Mover { x: 10, y: 20 });
    procesar(Mensaje::Escribir(String::from("hola")));
}
```

El `match` es **exhaustivo**: si añades una variante nueva a `Mensaje` y olvidas gestionarla en algún `match` del proyecto, **el compilador falla**. En C, añadir un valor a un `enum` y olvidar un `case` en algún `switch` es un bug silencioso que solo se descubre en tiempo de ejecución (o ni eso).

## 5.3 `Option<T>`: la solución de Rust al `NULL`

Rust **no tiene punteros nulos** en código seguro. En su lugar, cualquier valor que pueda "no estar presente" se modela con el enum de la librería estándar `Option<T>`:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

```rust
fn dividir(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None
    } else {
        Some(a / b)
    }
}

fn main() {
    let resultado = dividir(10.0, 2.0);

    match resultado {
        Some(valor) => println!("Resultado: {valor}"),
        None => println!("No se puede dividir por cero"),
    }

    // Formas más idiomáticas y concisas:
    if let Some(valor) = dividir(10.0, 0.0) {
        println!("Resultado: {valor}");
    } else {
        println!("Sin resultado");
    }

    let valor = dividir(10.0, 5.0).unwrap_or(0.0); // valor por defecto si es None
    println!("{valor}");
}
```

La diferencia crucial con C: **el compilador obliga a gestionar el caso `None`** antes de poder usar el valor interior. Olvidar comprobar un puntero nulo (la causa de innumerables crashes y CVEs en C) es, en Rust, directamente imposible de compilar — se tiene que desenvolver el `Option` explícitamente (con `match`, `if let`, `.unwrap()`, `?`, etc.).

## 5.4 Pattern matching avanzado

```rust
fn main() {
    let par = (3, -3);

    match par {
        (0, y) => println!("primer valor es cero, y = {y}"),
        (x, 0) => println!("segundo valor es cero, x = {x}"),
        (x, y) if x == -y => println!("son opuestos: {x} y {y}"), // match guard
        _ => println!("ningún patrón especial"),
    }

    // Desestructurar structs directamente
    struct Punto { x: i32, y: i32 }
    let p = Punto { x: 0, y: 7 };
    let Punto { x, y } = p;
    println!("{x} {y}");

    // @ bindings: capturar el valor Y comprobar un rango a la vez
    let edad = 15;
    match edad {
        n @ 0..=12 => println!("niño, edad {n}"),
        n @ 13..=17 => println!("adolescente, edad {n}"),
        n => println!("adulto, edad {n}"),
    }
}
```

## 5.5 Ejercicio

1. Define un enum `Forma` con variantes `Circulo(f64)` (radio), `Rectangulo(f64, f64)` y `Triangulo { base: f64, altura: f64 }`.
2. Implementa `fn area(&self) -> f64` usando `impl Forma` y `match` sobre `self`.
3. Crea un `Vec<Forma>` con varias formas distintas y, con un `for`, imprime el área de cada una.

**Siguiente →** [Tema 6: Traits y genéricos](06-traits-generics.md)
