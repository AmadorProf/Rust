# Tema 11 — Testing, módulos y organización de proyectos

## 11.1 Tests integrados en el lenguaje

A diferencia de C, donde probar código requiere un framework externo (Unity, CMocka, Check...) e integrarlo manualmente al build, en Rust el testing es parte de `cargo` desde el primer día.

```rust
fn suma(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)] // este módulo solo se compila al ejecutar `cargo test`
mod tests {
    use super::*; // importa todo lo del módulo padre (la función suma, aquí)

    #[test]
    fn suma_dos_positivos() {
        assert_eq!(suma(2, 3), 5);
    }

    #[test]
    fn suma_con_negativos() {
        assert_eq!(suma(-2, -3), -5);
    }

    #[test]
    #[should_panic(expected = "división por cero")]
    fn division_por_cero_hace_panic() {
        let _ = 10 / (5 - 5); // ejemplo ilustrativo, en la práctica se probaría una función real
    }
}
```

Ejecutar:

```bash
cargo test
```

```
running 2 tests
test tests::suma_dos_positivos ... ok
test tests::suma_con_negativos ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Macros de aserción más usadas: `assert!(condicion)`, `assert_eq!(a, b)`, `assert_ne!(a, b)`. Todas imprimen, si fallan, los valores reales comparados — no solo "assertion failed" como `assert()` de C.

### Tests de integración

Además de los tests unitarios (dentro de `src/`, junto al código), Cargo soporta tests de integración en un directorio `tests/` en la raíz del proyecto, que prueban la API pública del crate como lo haría un usuario externo:

```
mi_proyecto/
├── src/
│   └── lib.rs
└── tests/
    └── integracion.rs
```

### Doctests: ejemplos en la documentación que se ejecutan como tests

```rust
/// Suma dos números.
///
/// ```
/// let resultado = mi_crate::suma(2, 3);
/// assert_eq!(resultado, 5);
/// ```
pub fn suma(a: i32, b: i32) -> i32 {
    a + b
}
```

`cargo test` ejecuta también estos bloques de código de la documentación. Esto garantiza algo que en casi ningún otro lenguaje se garantiza automáticamente: **la documentación nunca queda desactualizada respecto al comportamiento real del código**, porque si el ejemplo deja de ser válido, el build falla.

## 11.2 Módulos: organizar el código

El sistema de módulos de Rust es jerárquico y explícito sobre qué es público y qué es privado (todo es privado por defecto, al contrario que en C donde todo símbolo no `static` es visible globalmente).

```rust
mod matematicas {
    pub fn suma(a: i32, b: i32) -> i32 {
        a + b
    }

    fn interna() -> i32 { // privada por defecto: no visible fuera de `matematicas`
        42
    }

    pub mod avanzadas { // submódulo anidado
        pub fn potencia(base: i32, exp: u32) -> i32 {
            base.pow(exp)
        }
    }
}

fn main() {
    println!("{}", matematicas::suma(2, 3));
    println!("{}", matematicas::avanzadas::potencia(2, 10));
}
```

### Separar módulos en ficheros

Para proyectos reales, cada módulo suele vivir en su propio fichero:

```
src/
├── main.rs
├── matematicas.rs          // contiene el módulo `matematicas`
└── matematicas/
    └── avanzadas.rs        // submódulo matematicas::avanzadas
```

```rust
// src/main.rs
mod matematicas; // busca matematicas.rs o matematicas/mod.rs

fn main() {
    println!("{}", matematicas::suma(2, 3));
}
```

```rust
// src/matematicas.rs
pub mod avanzadas;

pub fn suma(a: i32, b: i32) -> i32 {
    a + b
}
```

### `use`: traer nombres al ámbito actual

```rust
use std::collections::HashMap;       // sin esto, habría que escribir std::collections::HashMap siempre
use matematicas::avanzadas::potencia;

fn main() {
    let _mapa: HashMap<String, i32> = HashMap::new();
    println!("{}", potencia(2, 8));
}
```

## 11.3 Binarios vs librerías (`main.rs` vs `lib.rs`)

Un *crate* (unidad de compilación en Rust, equivalente a un paquete) puede ser:

- **Binario** (`src/main.rs`): produce un ejecutable. `fn main()` es el punto de entrada.
- **Librería** (`src/lib.rs`): produce código reutilizable por otros crates (o por el propio binario del mismo proyecto). No tiene `main()`, expone una API pública con `pub`.

Un patrón muy común en proyectos reales: poner toda la lógica en `lib.rs` (fácil de testear con tests de integración) y dejar `main.rs` como una capa delgada que solo parsea argumentos y llama a la librería. Esto es análogo a separar una librería estática (`.a`) de su programa cliente en C, pero resuelto dentro del mismo `Cargo.toml` sin pasos de build adicionales.

```toml
# Cargo.toml — un proyecto puede tener ambos a la vez
[package]
name = "mi_proyecto"

[[bin]]
name = "mi_proyecto"
path = "src/main.rs"
```

## 11.4 Workspaces: varios crates relacionados

Para proyectos grandes con varios crates relacionados (p. ej. una librería core + una CLI + un servidor web que la usan), Cargo soporta **workspaces**: un `Cargo.toml` raíz que agrupa varios paquetes, compartiendo el mismo `target/` y el mismo `Cargo.lock` (resolviendo versiones de dependencias una sola vez para todo el workspace).

```toml
# Cargo.toml raíz del workspace
[workspace]
members = ["core", "cli", "servidor"]
```

```
mi_workspace/
├── Cargo.toml       # define el workspace
├── core/
│   ├── Cargo.toml
│   └── src/lib.rs
├── cli/
│   ├── Cargo.toml
│   └── src/main.rs
└── servidor/
    ├── Cargo.toml
    └── src/main.rs
```

## 11.5 `cargo clippy` y `cargo fmt`

```bash
cargo fmt      # formatea todo el proyecto según el estándar (equivalente a clang-format, pero sin configuración que discutir)
cargo clippy   # linter oficial: detecta código no idiomático, ineficiencias y errores sutiles
```

`clippy` es especialmente valioso porque no solo señala estilo: detecta patrones que compilan pero son casi siempre errores lógicos (comparaciones redundantes, clones innecesarios, uso ineficiente de iteradores, etc.) — es habitual ejecutarlo en CI junto a `cargo test`.

## 11.6 Ejercicio

1. Convierte el ejercicio del Tema 7 (`parsear_edad`) en una librería: crea un proyecto con `cargo new --lib validaciones`, mueve la función a `src/lib.rs` marcándola `pub`, y añade tests unitarios dentro de un `mod tests` con `#[cfg(test)]` cubriendo al menos: una edad válida, una cadena no numérica, y un número fuera de rango.
2. Añade un doctest al comentario de documentación (`///`) de la función mostrando un ejemplo de uso correcto.
3. Ejecuta `cargo test`, `cargo clippy` y `cargo fmt` sobre el proyecto y corrige cualquier aviso que salga.

**Siguiente →** [Tema 12: Proyecto práctico — una CLI real](12-proyecto-practico-cli.md)
