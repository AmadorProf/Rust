# Tema 4 — Ownership, Borrowing y Lifetimes

> Este es el tema más importante de todo el tutorial. Si el Tema 1 explicó el "por qué", aquí vemos el "cómo" en código real, incluyendo los errores del compilador que vas a ver constantemente al empezar.

## 4.1 Ownership (propiedad)

Regla fundamental: **cada valor tiene un único dueño en cada momento**, y cuando el dueño sale de ámbito (*scope*), Rust llama automáticamente a `drop` para liberar los recursos (memoria, file handles, locks...). Es determinista: no hay que esperar a un GC, se sabe exactamente en qué línea se libera cada cosa (la llave de cierre `}` del scope).

```rust
fn main() {
    {
        let s = String::from("hola"); // s es dueño del String
        println!("{s}");
    } // aquí termina el scope: Rust llama a drop(s) automáticamente, se libera la memoria
    // println!("{s}"); // ERROR: s ya no existe aquí
}
```

### Move: la propiedad se transfiere, no se comparte por defecto

```rust
fn main() {
    let s1 = String::from("hola");
    let s2 = s1; // la propiedad de "hola" se MUEVE de s1 a s2

    println!("{s2}");
    // println!("{s1}"); // ERROR: "value borrowed here after move"
}
```

Esto es muy distinto de C++, donde `s2 = s1` haría una copia completa por defecto (a menos que sea un `std::move` explícito). En Rust, para tipos que gestionan memoria en el heap (`String`, `Vec`, `Box`...), la asignación por defecto **mueve** la propiedad, dejando la variable original inválida. Esto evita el error clásico del **double free**: si ambas variables fueran "dueñas" del mismo dato, al salir de scope ambas intentarían liberarlo, liberando memoria dos veces (un bug de seguridad grave en C). Rust lo resuelve asegurando que solo una variable es dueña a la vez.

Si realmente se quiere una copia independiente, hay que pedirla explícitamente:

```rust
let s1 = String::from("hola");
let s2 = s1.clone(); // copia profunda explícita, ahora ambas son válidas
println!("{s1} {s2}");
```

### `Copy` vs `Move`: tipos simples en la pila

Los tipos escalares simples (`i32`, `f64`, `bool`, `char`, tuplas de estos) implementan el trait `Copy`: se copian automáticamente en la pila (como en C), no se "mueven". Por eso esto funciona sin error:

```rust
let x = 5;
let y = x; // copia, no move
println!("{x} {y}"); // ambas válidas
```

La regla general: si un tipo solo vive en la pila y copiarlo es barato y seguro, implementa `Copy` y se comporta "como en C". Si gestiona recursos en el heap (memoria dinámica, file handles...), se mueve.

### Ownership y funciones

Pasar un valor a una función también transfiere la propiedad, salvo que se pase por referencia:

```rust
fn tomar_ownership(s: String) {
    println!("{s}");
} // aquí se libera s

fn main() {
    let texto = String::from("hola");
    tomar_ownership(texto);
    // println!("{texto}"); // ERROR: texto fue movido a la función
}
```

## 4.2 Borrowing (préstamo): referencias sin transferir propiedad

En vez de mover la propiedad cada vez (lo cual sería muy incómodo), se puede **prestar** una referencia con `&`:

```rust
fn calcular_longitud(s: &String) -> usize {
    s.len()
} // s es una referencia, no se libera nada al terminar

fn main() {
    let texto = String::from("hola mundo");
    let len = calcular_longitud(&texto); // prestamos una referencia
    println!("{texto} tiene {len} caracteres"); // texto sigue siendo válido aquí
}
```

### La regla de oro del borrow checker

En un mismo ámbito, para un mismo valor:

- Puede haber **cualquier número de referencias inmutables** (`&T`) simultáneas, **o**
- Puede haber **exactamente una referencia mutable** (`&mut T`), **pero nunca ambas combinaciones a la vez**.

```rust
fn main() {
    let mut x = 5;

    let r1 = &x;
    let r2 = &x; // OK: varias referencias inmutables a la vez
    println!("{r1} {r2}");

    let r3 = &mut x; // OK ahora, porque r1 y r2 ya no se usan más (su "vida" terminó)
    *r3 += 1;
    println!("{r3}");
}
```

Ejemplo de **error real** del compilador (esto es exactamente lo que verás al empezar):

```rust
fn main() {
    let mut x = 5;
    let r1 = &x;
    let r2 = &mut x; // ERROR
    println!("{r1} {r2}");
}
```

```
error[E0502]: cannot borrow `x` as mutable because it is also borrowed as immutable
```

Esta regla, verificada en compilación, es **exactamente** lo que elimina los data races en concurrencia (Tema 9): si dos hilos tuvieran acceso simultáneo de escritura, o uno de lectura y otro de escritura sin sincronización, el compilador lo detecta antes de que el programa exista como binario.

### Referencias colgantes (dangling references): imposible en Rust

```rust
fn referencia_invalida() -> &String { // ERROR de compilación
    let s = String::from("hola");
    &s
} // s se libera aquí, la referencia que intentamos devolver ya no apunta a nada válido
```

```
error[E0106]: missing lifetime specifier
```

El compilador rechaza este código. En C, el equivalente (devolver un puntero a una variable local) compila sin ningún aviso y es undefined behavior en tiempo de ejecución.

## 4.3 Lifetimes (tiempos de vida)

Un *lifetime* es, conceptualmente, "durante qué región del código es válida una referencia". La mayoría de las veces el compilador lo infiere solo (*lifetime elision*) y nunca se escribe explícitamente. Se vuelve explícito cuando el compilador no puede deducirlo sin ayuda, típicamente en funciones que devuelven una referencia derivada de más de un parámetro:

```rust
fn mas_largo<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("hola");
    let s2 = String::from("mundo!!");
    println!("{}", mas_largo(&s1, &s2));
}
```

La anotación `'a` no cambia cuánto vive nada: **solo le dice al compilador** que el valor devuelto vivirá, como máximo, tanto como el más corto de los dos *inputs* — es una restricción que el compilador verifica, no una instrucción de "mantener vivo". Si en el código que llama a `mas_largo` se intentara usar el resultado después de que `s1` o `s2` dejaran de ser válidos, sería un error de compilación.

`'static` es un lifetime especial: significa "válido durante todo el programa" (p. ej. los string literals `&'static str` que vimos en el Tema 3, que viven embebidos en el binario).

## 4.4 Comparación directa con la gestión manual en C

```c
// C: el programador es 100% responsable de la disciplina
char *crear_saludo() {
    char *s = malloc(20);
    strcpy(s, "Hola");
    return s; // el que llama DEBE recordar hacer free(s)
}

int main() {
    char *saludo = crear_saludo();
    printf("%s\n", saludo);
    free(saludo);
    // free(saludo); // double free si se llama dos veces -> UB, posible vulnerabilidad
    // printf("%s\n", saludo); // use-after-free si se usa tras el free -> UB
}
```

```rust
// Rust: el compilador verifica la disciplina
fn crear_saludo() -> String {
    String::from("Hola")
}

fn main() {
    let saludo = crear_saludo();
    println!("{saludo}");
} // drop(saludo) se inserta automáticamente aquí, exactamente una vez, garantizado
```

No hay `free` manual, no hay riesgo de double-free, no hay riesgo de use-after-free, y no hay recolector de basura corriendo en segundo plano: todo se resuelve en tiempo de compilación insertando las llamadas a `drop` en los puntos exactos donde el ownership termina.

## 4.5 Errores comunes de principiante (y cómo pensarlos)

- **"Cannot move out of borrowed content"**: intentaste tomar propiedad de algo a través de una referencia. Solución habitual: clonar (`.clone()`) o rediseñar para no necesitar la propiedad, solo leer/escribir vía referencia.
- **"Cannot borrow as mutable more than once"**: tienes dos `&mut` activas a la vez, o una `&mut` y una `&` a la vez. Solución: acortar el scope de la primera referencia (a menudo basta con un bloque `{}` o reordenar el código) antes de crear la segunda.
- **"Borrowed value does not live long enough"**: estás devolviendo o guardando una referencia a algo que se destruye antes. Solución: o bien devuelve el valor con ownership (no una referencia), o bien usa un lifetime explícito si el valor sí vive lo suficiente pero el compilador necesita la pista.

La experiencia habitual al empezar es "pelearse con el borrow checker" varias veces al día durante las primeras semanas. Es normal y esperado — es la curva de aprendizaje mencionada en el Tema 1, y se reduce drásticamente con la práctica.

## 4.6 Ejercicio

1. Escribe una función `fn primera_palabra(s: &str) -> &str` que devuelva la primera palabra de una frase (hasta el primer espacio, o la frase entera si no hay espacios), usando `s.find(' ')` y slicing (`&s[0..i]`).
2. Intenta provocar deliberadamente un error de borrow checker: crea un `Vec<i32>`, guarda una referencia a su primer elemento con `.first()`, y luego intenta hacer `vec.push(...)` mientras esa referencia sigue viva. Lee el mensaje de error completo que da el compilador — está diseñado para ser pedagógico.

**Siguiente →** [Tema 5: Structs, enums y pattern matching](05-structs-enums-pattern-matching.md)
