# Tema 7 — Manejo de errores

## 7.1 Dos categorías de error en Rust

Rust distingue explícitamente entre:

- **Errores recuperables**: situaciones esperables (un fichero no existe, una entrada de usuario es inválida, falla una petición de red). Se modelan con `Result<T, E>`.
- **Errores irrecuperables**: bugs del programa que no deberían ocurrir si el código es correcto (un índice fuera de rango, una invariante rota). Se gestionan con `panic!`, que aborta el hilo actual (y por defecto el proceso).

Esto es una diferencia filosófica importante frente a C, donde todo son códigos de retorno (fácilmente ignorables) o `errno` (fácil de olvidar comprobar), y frente a lenguajes con excepciones, donde cualquier función puede lanzar cualquier excepción en silencio sin que la firma lo indique.

## 7.2 `Result<T, E>`

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

```rust
use std::fs::File;

fn main() {
    let archivo = File::open("config.txt");

    match archivo {
        Ok(f) => println!("Archivo abierto: {f:?}"),
        Err(e) => println!("Error al abrir: {e}"),
    }
}
```

La clave: **el compilador obliga a gestionar el `Err`** (o al menos a reconocer explícitamente que se ignora). No hay forma de "olvidarse" de comprobar si una operación que puede fallar, falló — a diferencia de C, donde ignorar el valor de retorno de una función compila perfectamente.

### El operador `?`: propagar errores sin boilerplate

```rust
use std::fs::File;
use std::io::{self, Read};

fn leer_archivo(ruta: &str) -> Result<String, io::Error> {
    let mut f = File::open(ruta)?;      // si falla, retorna el Err inmediatamente
    let mut contenido = String::new();
    f.read_to_string(&mut contenido)?;  // igual aquí
    Ok(contenido)
}

fn main() {
    match leer_archivo("config.txt") {
        Ok(contenido) => println!("Contenido:\n{contenido}"),
        Err(e) => eprintln!("No se pudo leer el archivo: {e}"),
    }
}
```

`?` es azúcar sintáctico: si el `Result` es `Err`, retorna ese error inmediatamente desde la función actual (convirtiéndolo con `From` si hace falta); si es `Ok`, extrae el valor y continúa. Es el equivalente, pero verificado por tipos, a lanzar una excepción — salvo que aquí está **en la firma de la función** (`-> Result<T, E>`), así que quien llama a la función sabe, sin leer el cuerpo, que puede fallar.

## 7.3 `unwrap()` y `expect()`: cuándo (no) usarlos

```rust
let archivo = File::open("config.txt").unwrap();   // panic con mensaje genérico si falla
let archivo = File::open("config.txt").expect("No se encontró config.txt"); // panic con mensaje propio
```

`unwrap`/`expect` convierten un `Err`/`None` en un `panic!` inmediato. Son útiles para:
- Prototipos y ejemplos rápidos.
- Casos donde un fallo es realmente imposible por invariantes del programa (y aun así es mejor usar `expect` con un mensaje explicando por qué se asume que no puede fallar).

**No deben usarse** en código de producción para errores esperables (entrada de usuario, I/O, red): ahí corresponde propagar con `?` y gestionar el error en un nivel superior (log, reintento, mensaje al usuario).

## 7.4 `panic!`: errores irrecuperables

```rust
fn dividir(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("división por cero"); // aborta el hilo actual
    }
    a / b
}
```

Por defecto, un `panic!` "deshace la pila" (*unwinding*, liberando los recursos de cada scope igual que un `drop` normal) y termina el hilo o proceso. Se puede configurar para abortar inmediatamente (`panic = "abort"` en `Cargo.toml`), útil en sistemas embebidos donde el *unwinding* tiene coste de binario.

A diferencia de un `segfault` en C (comportamiento indefinido, puede corromper memoria silenciosamente), un `panic!` en Rust es un fallo **controlado**: se sabe exactamente dónde ocurrió, con mensaje y traza, y la memoria no queda en un estado corrupto.

## 7.5 Errores personalizados

Para una aplicación real conviene definir tipos de error propios, normalmente como un enum:

```rust
use std::fmt;

#[derive(Debug)]
enum ErrorApp {
    NoEncontrado(String),
    Invalido { campo: String, razon: String },
}

impl fmt::Display for ErrorApp {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ErrorApp::NoEncontrado(id) => write!(f, "no encontrado: {id}"),
            ErrorApp::Invalido { campo, razon } => write!(f, "campo '{campo}' inválido: {razon}"),
        }
    }
}

impl std::error::Error for ErrorApp {}

fn buscar_usuario(id: &str) -> Result<String, ErrorApp> {
    if id.is_empty() {
        return Err(ErrorApp::Invalido {
            campo: "id".into(),
            razon: "no puede estar vacío".into(),
        });
    }
    Err(ErrorApp::NoEncontrado(id.to_string()))
}
```

### En la práctica: `thiserror` y `anyhow`

Escribir `impl Display` y `impl Error` a mano para cada tipo de error es repetitivo. El ecosistema ofrece dos crates estándar de facto:

- **`thiserror`**: para librerías, genera `Display`/`Error` automáticamente a partir de atributos.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum ErrorApp {
    #[error("no encontrado: {0}")]
    NoEncontrado(String),
    #[error("campo '{campo}' inválido: {razon}")]
    Invalido { campo: String, razon: String },
}
```

- **`anyhow`**: para aplicaciones (binarios), cuando no importa tanto el tipo exacto del error sino propagarlo con contexto hasta `main`.

```rust
use anyhow::{Context, Result};

fn leer_config() -> Result<String> {
    std::fs::read_to_string("config.toml")
        .context("no se pudo leer config.toml")
}

fn main() -> Result<()> {
    let config = leer_config()?;
    println!("{config}");
    Ok(())
}
```

Nótese que `main` puede devolver `Result`: si retorna `Err`, el proceso termina con código de salida distinto de 0 e imprime el error — muy útil para CLIs (ver Tema 12).

## 7.6 Comparación con C

| | C | Rust |
|---|---|---|
| Señalar fallo | Valor de retorno especial (`-1`, `NULL`) + `errno` global | `Result<T, E>` explícito en el tipo de retorno |
| Ignorar un fallo | Compila sin aviso | Produce un *warning* (`unused Result`) salvo que se gestione explícitamente |
| Propagar un fallo | `if (fallo) return -1;` repetido en cada nivel | Operador `?`, una línea |
| Error "fatal" | `abort()`/UB si hay corrupción de memoria | `panic!`, controlado, sin corrupción de memoria |
| Tipado del error | Ninguno (entero/puntero genérico) | Tipo concreto (`enum`, `thiserror`, etc.), verificado por el compilador |

## 7.7 Ejercicio

1. Escribe `fn parsear_edad(s: &str) -> Result<u8, String>` que convierta un `&str` a `u8` usando `.parse()`, devolviendo un mensaje de error propio si falla el parseo o si el número es mayor de 120.
2. Escribe una función `main` que lea varias cadenas de un `Vec<&str>`, llame a `parsear_edad` para cada una, y use `match` para imprimir el resultado o el error de cada una (sin que un error detenga el procesamiento de las demás).
3. (Extra) Reescribe el ejercicio usando el crate `anyhow` para la función `main`, devolviendo `anyhow::Result<()>`.

**Siguiente →** [Tema 8: Colecciones e iteradores](08-colecciones-e-iteradores.md)
