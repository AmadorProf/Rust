# Tema 8 — Colecciones e iteradores

## 8.1 `Vec<T>`: el array dinámico

Equivalente a `std::vector` de C++ o a una lista dinámica de C hecha a mano con `malloc`/`realloc`, pero gestionado por el sistema de ownership: no hay que liberar memoria manualmente ni preocuparse por *reallocations* colgantes.

```rust
fn main() {
    let mut numeros: Vec<i32> = Vec::new();
    numeros.push(1);
    numeros.push(2);
    numeros.push(3);

    let numeros2 = vec![1, 2, 3]; // macro de conveniencia, equivalente

    println!("{:?}", numeros);
    println!("Primero: {:?}", numeros.first()); // Option<&i32>, no "unsafe" como en C
    println!("Elemento 1: {}", numeros[1]);     // panic controlado si está fuera de rango

    for n in &numeros { // iterar por referencia: no consume el vector
        println!("{n}");
    }
}
```

Acceder con `numeros[10]` fuera de rango hace `panic!` (controlado, con mensaje). Si se prefiere gestionar el caso explícitamente sin arriesgar un panic, se usa `.get(10)`, que devuelve `Option<&i32>`.

## 8.2 `HashMap<K, V>`

Equivalente a un diccionario de Python o un `std::unordered_map` de C++. En C no existe en el lenguaje: hay que implementarlo o traer una librería externa.

```rust
use std::collections::HashMap;

fn main() {
    let mut edades: HashMap<String, u8> = HashMap::new();
    edades.insert(String::from("Ana"), 30);
    edades.insert(String::from("Luis"), 25);

    match edades.get("Ana") {
        Some(edad) => println!("Ana tiene {edad} años"),
        None => println!("No encontrada"),
    }

    // entry API: patrón muy idiomático para "insertar si no existe, o modificar"
    let contador = edades.entry(String::from("Ana")).or_insert(0);
    *contador += 1;

    for (nombre, edad) in &edades {
        println!("{nombre}: {edad}");
    }
}
```

## 8.3 Otras colecciones de la librería estándar

| Tipo | Uso |
|---|---|
| `VecDeque<T>` | Cola doblemente terminada, eficiente en ambos extremos |
| `HashSet<T>` | Conjunto sin orden, basado en hashing |
| `BTreeMap<K, V>` / `BTreeSet<T>` | Como HashMap/HashSet pero ordenados por clave |
| `[T; N]` | Array de tamaño fijo conocido en compilación |
| `&[T]` (slice) | Vista prestada sobre una secuencia contigua, sin ownership |

### Slices: una de las ideas más potentes de Rust

Un *slice* (`&[T]` o `&str` para texto) es una referencia a una parte contigua de una colección, sin copiar los datos y sin tomar ownership — pero con los límites verificados en tiempo de ejecución (a diferencia de pasar un `puntero + longitud` en C, donde nada impide pasar una longitud incorrecta).

```rust
fn main() {
    let numeros = vec![10, 20, 30, 40, 50];
    let parte: &[i32] = &numeros[1..3]; // [20, 30], sin copiar
    println!("{parte:?}");

    fn suma(slice: &[i32]) -> i32 {
        slice.iter().sum()
    }
    println!("{}", suma(&numeros)); // un Vec se puede pasar como &[T] directamente
    println!("{}", suma(parte));
}
```

Esto permite escribir funciones que aceptan tanto arrays como vectores (o partes de ellos) sin genéricos, sin copiar y sin riesgo de desbordar límites.

## 8.4 Iteradores: la forma idiomática de procesar datos

En C, procesar una colección significa escribir bucles `for` manuales con índices. En Rust, el *trait* `Iterator` permite encadenar transformaciones de forma declarativa, y gracias a las *zero-cost abstractions* (Tema 1.4) el código compilado es tan eficiente como el bucle manual equivalente.

```rust
fn main() {
    let numeros = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let pares_al_cuadrado: Vec<i32> = numeros
        .iter()
        .filter(|&&n| n % 2 == 0) // quedarse solo con los pares
        .map(|&n| n * n)          // elevar al cuadrado
        .collect();               // materializar en un nuevo Vec

    println!("{pares_al_cuadrado:?}"); // [4, 16, 36, 64, 100]

    let suma: i32 = numeros.iter().sum();
    let maximo = numeros.iter().max();
    let hay_mayor_de_5 = numeros.iter().any(|&n| n > 5);
    let todos_positivos = numeros.iter().all(|&n| n > 0);

    println!("suma={suma}, max={maximo:?}, hay_mayor_de_5={hay_mayor_de_5}, todos_positivos={todos_positivos}");

    // enumerate: índice + valor, sin gestionar un contador manualmente
    for (i, n) in numeros.iter().enumerate() {
        if i < 3 {
            println!("índice {i}: {n}");
        }
    }
}
```

### Iteradores son "lazy" (perezosos)

```rust
let it = numeros.iter().map(|n| {
    println!("procesando {n}");
    n * 2
});
// nada se ha ejecutado todavía: los adaptadores solo describen la cadena de transformación
let resultado: Vec<i32> = it.collect(); // aquí sí se ejecuta todo, de una sola pasada
```

Esto permite encadenar muchas operaciones sin crear colecciones intermedias en memoria: `filter().map().take(5)` sobre un iterador de un millón de elementos solo procesa lo necesario, no los 1.000.000 completos si `take(5)` ya se satisface antes.

### `for` vs iteradores: ¿cuál usar?

Ambos compilan a código equivalente. La convención en la comunidad es:
- `for` cuando hay efectos secundarios simples (imprimir, mutar estado externo) y la lógica es lineal.
- Cadenas de iteradores (`.filter().map().collect()`) cuando se transforma una colección en otra — es más declarativo y suele ser más fácil de leer una vez acostumbrado.

## 8.5 Comparación con C

| | C | Rust |
|---|---|---|
| Array dinámico | `malloc`/`realloc` manual, tamaño gestionado a mano | `Vec<T>`, gestión automática y segura |
| Diccionario | No existe, hay que implementarlo o usar librería externa | `HashMap<K, V>` en la librería estándar |
| Acceso fuera de rango | Comportamiento indefinido (UB) | `panic!` controlado, o `Option` con `.get()` |
| Procesar una colección | Bucle `for` manual con índices | Iteradores encadenables, "zero-cost" |
| Pasar "un trozo" de un array | Puntero + longitud por separado (fácil de desincronizar) | Slice `&[T]`, longitud va integrada y verificada |

## 8.6 Ejercicio

1. Dado `let palabras = vec!["manzana", "pera", "kiwi", "uva", "sandía"];`, usa iteradores para obtener un `Vec<&str>` solo con las palabras de más de 4 letras, en mayúsculas (usa `.to_uppercase()`, ojo: devuelve `String`, no `&str`).
2. Usa un `HashMap<char, u32>` para contar cuántas veces aparece cada letra en la cadena `"abracadabra"`, usando `.chars()` y el patrón `entry().or_insert(0)`.
3. (Extra) Calcula el mismo resultado del punto 2 pero exclusivamente con una cadena de iteradores (`fold`), sin bucles `for` explícitos.

**Siguiente →** [Tema 9: Concurrencia](09-concurrencia.md)
