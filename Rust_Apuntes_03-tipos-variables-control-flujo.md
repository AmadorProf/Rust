# Tema 3 — Tipos, variables y control de flujo

## 3.1 Variables: inmutables por defecto

A diferencia de C, en Rust las variables son **inmutables por defecto**. Esto no es una limitación arbitraria: es una decisión de diseño que reduce bugs (una variable que no debería cambiar, no puede cambiar) y ayuda al compilador a razonar sobre el código.

```rust
fn main() {
    let x = 5;
    // x = 6; // ERROR de compilación: no se puede asignar dos veces a una variable inmutable
    println!("{x}");

    let mut y = 5; // mut la hace mutable, explícito y buscable
    y = 6;
    println!("{y}");
}
```

### Shadowing

Se puede "re-declarar" una variable con `let`, ocultando (*shadowing*) la anterior. Es distinto de mutar: se crea una variable nueva, que puede incluso tener otro tipo.

```rust
let x = "42";
let x: i32 = x.parse().unwrap(); // ahora x es un i32, otra variable con el mismo nombre
```

## 3.2 Tipos escalares

| Tipo | Descripción | Equivalente en C |
|---|---|---|
| `i8,i16,i32,i64,i128,isize` | Enteros con signo | `int8_t`... |
| `u8,u16,u32,u64,u128,usize` | Enteros sin signo | `uint8_t`... |
| `f32,f64` | Flotantes | `float`, `double` |
| `bool` | `true`/`false` | `_Bool` / `int` |
| `char` | Un carácter **Unicode** (4 bytes) | `char` (1 byte, solo ASCII/latin1) |

`usize`/`isize` tienen el tamaño del puntero de la plataforma (igual que `size_t` en C) y son el tipo que se usa para indexar arrays/vectores.

Por defecto, si no se especifica, un entero literal es `i32` y un flotante es `f64`.

```rust
let a: u8 = 255;
let b = 3.14; // f64 por defecto
let c: char = '🦀'; // char es Unicode, no solo ASCII
```

**Importante — overflow en debug vs release:** en modo `debug`, una suma que desborda el tipo (`u8` + `u8` > 255) hace **panic** (aborta el programa con un error claro). En modo `release`, por rendimiento, el comportamiento por defecto es *wrapping* (da la vuelta, como en C). Si se necesita un comportamiento explícito y controlado, existen métodos como `checked_add`, `wrapping_add`, `saturating_add`.

```rust
let x: u8 = 250;
let y = x.checked_add(10); // devuelve None en lugar de desbordar silenciosamente
println!("{y:?}"); // None
```

## 3.3 Tuplas y arrays

```rust
// Tupla: longitud fija, tipos pueden ser distintos
let punto: (i32, i32, f64) = (3, 4, 5.0);
println!("{}", punto.0); // acceso por índice

// Array: longitud fija, todos del mismo tipo, tamaño conocido en compilación
let numeros: [i32; 5] = [1, 2, 3, 4, 5];
println!("{}", numeros[0]);
```

A diferencia de C, acceder a `numeros[10]` (fuera de rango) **no es undefined behavior**: Rust comprueba el límite en tiempo de ejecución y hace `panic!` con un mensaje claro. Esto tiene un pequeño coste en tiempo de ejecución, pero elimina una categoría entera de vulnerabilidades (buffer overreads).

## 3.4 `String` vs `&str`

Esta es una de las primeras cosas que sorprende a quien viene de C. Hay dos tipos de cadena principales:

- **`&str`**: una referencia a una secuencia de bytes UTF-8 válida, de tamaño fijo, normalmente inmutable. Un *string literal* (`"hola"`) es de tipo `&'static str`.
- **`String`**: una cadena propia, mutable, asignada en el heap (como un `Vec<u8>` con garantías de UTF-8 válido). Equivalente conceptualmente a `std::string` de C++, muy distinto del `char*` de C (que no valida codificación ni lleva su longitud).

```rust
let saludo: &str = "hola";        // vive en el binario, referencia estática
let mut nombre: String = String::from("Ana"); // en el heap, se puede modificar
nombre.push_str(" García");
println!("{saludo} {nombre}");
```

La relación es parecida a `const char*` vs `std::string` en C++: casi todas las funciones que reciben `&str` también aceptan un `&String` gracias a *deref coercion*.

## 3.5 Control de flujo

### `if`/`else` (es una expresión, no una sentencia)

```rust
let n = 7;
let paridad = if n % 2 == 0 { "par" } else { "impar" };
println!("{paridad}");
```

No hace falta `return` explícito: el valor de la última línea de un bloque es el valor del bloque (como en Ruby o en las expresiones de Lisp). Esto es constante en todo el lenguaje, no solo en `if`.

### `loop`, `while`, `for`

```rust
// loop: bucle infinito explícito, puede devolver un valor con `break valor`
let resultado = loop {
    break 42;
};

// while: igual que en C
let mut contador = 0;
while contador < 5 {
    contador += 1;
}

// for: itera sobre algo que implementa IntoIterator — no hay "for con índice" clásico de C
for i in 0..5 {        // rango 0..5 (excluye el 5)
    println!("{i}");
}

for i in 0..=5 {       // rango inclusivo
    println!("{i}");
}

let frutas = ["manzana", "pera", "uva"];
for fruta in frutas.iter() {
    println!("{fruta}");
}
```

No existe el clásico `for (int i = 0; i < n; i++)` de C como forma idiomática: se prefiere iterar sobre rangos o colecciones directamente, lo cual elimina errores de "off-by-one" y de indexación fuera de rango.

### `match`: el `switch` con superpoderes

`match` es exhaustivo: el compilador obliga a cubrir todos los casos posibles (o usar `_` como comodín). Esto elimina el bug clásico de un `switch` en C al que se le olvida un `case` nuevo tras añadir un valor a un `enum`.

```rust
let numero = 3;
match numero {
    1 => println!("uno"),
    2 | 3 => println!("dos o tres"), // varios patrones
    4..=9 => println!("entre 4 y 9"), // rango
    _ => println!("otro"),            // obligatorio si no se cubren todos los casos
}
```

`match` también es una expresión:

```rust
let descripcion = match numero {
    n if n < 0 => "negativo",   // guard
    0 => "cero",
    _ => "positivo",
};
```

## 3.6 Funciones

```rust
fn suma(a: i32, b: i32) -> i32 {
    a + b // sin `return`: es la última expresión del bloque
}

fn main() {
    println!("{}", suma(2, 3));
}
```

A diferencia de C, los tipos de los parámetros y del retorno son **obligatorios** (no hay inferencia en firmas de función, solo dentro del cuerpo), lo que hace que las firmas sirvan de documentación fiable y verificada.

## 3.7 Ejercicio

Escribe una función `fn clasificar(n: i32) -> &'static str` que use `match` para devolver `"negativo"`, `"cero"`, `"par positivo"` o `"impar positivo"` según el valor de `n`, y pruébala con varios valores desde `main` usando un `for` sobre un array.

(No te preocupes todavía por el `&'static str` del tipo de retorno — se explica en el Tema 4, es un *lifetime*.)

**Siguiente →** [Tema 4: Ownership, Borrowing y Lifetimes](04-ownership-borrowing-lifetimes.md)
