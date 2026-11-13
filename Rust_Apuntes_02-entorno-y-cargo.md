# Tema 2 — Entorno de trabajo: Cargo, toolchain y primer programa

## 2.1 Instalación

Rust se instala con **rustup**, el instalador/gestor de versiones oficial (equivalente a `nvm` para Node o `pyenv` para Python):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Esto instala:
- `rustc`: el compilador.
- `cargo`: el gestor de proyectos/dependencias/build.
- `rustup`: el gestor de toolchains (versiones stable/beta/nightly, targets de compilación cruzada).

Comprobar instalación:

```bash
rustc --version
cargo --version
```

Actualizar a la última versión estable:

```bash
rustup update
```

## 2.2 Cargo: el corazón del día a día

Cargo es a Rust lo que `npm`+`webpack` son a Node, o lo que `pip`+`setuptools` son a Python, pero integrado en una sola herramienta coherente desde el primer commit del proyecto.

### Crear un proyecto nuevo

```bash
cargo new saludo
cd saludo
```

Esto genera:

```
saludo/
├── Cargo.toml   # metadatos del proyecto + dependencias ("crates")
└── src/
    └── main.rs  # punto de entrada
```

`Cargo.toml` (análogo a `package.json` o `pyproject.toml`):

```toml
[package]
name = "saludo"
version = "0.1.0"
edition = "2021"

[dependencies]
```

### Comandos esenciales

| Comando | Qué hace |
|---|---|
| `cargo build` | Compila en modo debug (`target/debug/`) |
| `cargo build --release` | Compila con optimizaciones (`target/release/`) |
| `cargo run` | Compila (si hace falta) y ejecuta |
| `cargo check` | Comprueba que compila **sin** generar binario (mucho más rápido, ideal para iterar) |
| `cargo test` | Ejecuta los tests |
| `cargo doc --open` | Genera y abre la documentación del proyecto y sus dependencias |
| `cargo add <crate>` | Añade una dependencia a `Cargo.toml` (ej: `cargo add serde`) |
| `cargo fmt` | Formatea el código según el estándar oficial |
| `cargo clippy` | Linter oficial: detecta patrones no idiomáticos o ineficientes |

`cargo check` es el comando que más se usa mientras se escribe código: no genera el binario final, solo valida tipos y *borrow checker*, por lo que es mucho más rápido que `build`.

## 2.3 Primer programa

`src/main.rs`:

```rust
fn main() {
    println!("Hola, Rust!");
}
```

- `fn main()`: punto de entrada, igual que en C.
- `println!`: no es una función, es una **macro** (de ahí el `!`). Las macros en Rust se expanden en tiempo de compilación y permiten cosas que una función normal no podría, como comprobar en compilación que el número de argumentos coincide con los `{}` del formato.

Ejecutar:

```bash
cargo run
```

```
Hola, Rust!
```

## 2.4 Comparación rápida con el flujo de C

| Paso | C | Rust |
|---|---|---|
| Crear proyecto | `mkdir`, `touch main.c`, escribir `Makefile` a mano | `cargo new` |
| Compilar | `gcc main.c -o main` | `cargo build` |
| Gestionar dependencias | vendoring manual, `pkg-config`, submódulos git | `cargo add` + `Cargo.toml` |
| Ejecutar tests | Framework externo (Unity, CMocka...) + integración manual | `cargo test` integrado desde el día 1 |
| Documentación | Doxygen externo | `cargo doc`, comentarios `///` generan docs con ejemplos ejecutables |

## 2.5 `rust-analyzer` y el editor

Para trabajar cómodamente se recomienda el plugin **rust-analyzer** (disponible para VS Code, Neovim, JetBrains...). Da autocompletado, "go to definition", inline type hints y, sobre todo, **muestra los errores del borrow checker en tiempo real** mientras escribes, lo cual acelera mucho el aprendizaje de los temas 4 y siguientes.

## 2.6 Ejercicio

1. Crea un proyecto `cargo new ejercicio1`.
2. Modifica `main.rs` para que imprima tu nombre y la versión de Rust instalada usando `env!("CARGO_PKG_VERSION")` (no te preocupes si no entiendes bien qué es `env!` todavía — solo compruébalo).
3. Ejecútalo con `cargo run` y luego compílalo en modo release con `cargo build --release` y compara el tamaño del binario (`ls -la target/debug/ejercicio1 target/release/ejercicio1`).

**Siguiente →** [Tema 3: Tipos, variables y control de flujo](03-tipos-variables-control-flujo.md)
