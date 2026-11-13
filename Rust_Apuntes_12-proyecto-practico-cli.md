# Tema 12 — Proyecto práctico: una CLI real de principio a fin

> Objetivo: construir **`tareas`**, una pequeña CLI de gestión de tareas (tipo *todo list*) que persiste en un fichero JSON, usando prácticamente todo lo visto en los temas anteriores: structs/enums, `Result`/`?`, traits (`derive`), colecciones, módulos y un par de crates del ecosistema. Es el tipo de proyecto real para el que Rust es una opción habitual frente a Python o Node (Tema 1.7): un binario único, rápido de arrancar, sin runtime que instalar en la máquina de destino.

## 12.1 Crear el proyecto y añadir dependencias

```bash
cargo new tareas
cd tareas
cargo add clap --features derive   # parseo de argumentos de línea de comandos
cargo add serde --features derive  # (de)serialización
cargo add serde_json               # formato JSON concreto
cargo add anyhow                   # manejo de errores ergonómico en la app (Tema 7)
```

`Cargo.toml` resultante (versiones exactas pueden variar):

```toml
[package]
name = "tareas"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
```

## 12.2 Modelar los datos

```rust
// src/tarea.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Tarea {
    pub id: u32,
    pub descripcion: String,
    pub completada: bool,
}

impl Tarea {
    pub fn nueva(id: u32, descripcion: String) -> Self {
        Tarea { id, descripcion, completada: false }
    }
}
```

`#[derive(Serialize, Deserialize)]` de `serde` genera automáticamente el código para convertir `Tarea` a/desde JSON — equivalente a lo que en C habría que escribir a mano (o con una librería como `cJSON`) parseando campo por campo y gestionando errores de formato manualmente.

## 12.3 Persistencia: leer y escribir el fichero

```rust
// src/almacen.rs
use crate::tarea::Tarea;
use anyhow::{Context, Result};
use std::path::Path;

const ARCHIVO: &str = "tareas.json";

pub fn cargar() -> Result<Vec<Tarea>> {
    if !Path::new(ARCHIVO).exists() {
        return Ok(Vec::new()); // primera ejecución: no hay nada que cargar todavía
    }
    let contenido = std::fs::read_to_string(ARCHIVO)
        .context("no se pudo leer tareas.json")?;
    let tareas = serde_json::from_str(&contenido)
        .context("tareas.json tiene un formato inválido")?;
    Ok(tareas)
}

pub fn guardar(tareas: &[Tarea]) -> Result<()> {
    let contenido = serde_json::to_string_pretty(tareas)
        .context("no se pudo serializar las tareas")?;
    std::fs::write(ARCHIVO, contenido)
        .context("no se pudo escribir tareas.json")?;
    Ok(())
}
```

Nótese el uso de `?` y `.context(...)` (Tema 7): cada posible fallo de I/O se propaga con un mensaje claro de **en qué paso** ocurrió, sin un solo `if (error) { ... }` repetido manualmente como en C.

## 12.4 La lógica de negocio (pensada para testear fácilmente)

```rust
// src/lib.rs
pub mod tarea;
pub mod almacen;

use anyhow::{bail, Result};
use tarea::Tarea;

pub fn añadir(tareas: &mut Vec<Tarea>, descripcion: String) -> u32 {
    let nuevo_id = tareas.iter().map(|t| t.id).max().unwrap_or(0) + 1;
    tareas.push(Tarea::nueva(nuevo_id, descripcion));
    nuevo_id
}

pub fn completar(tareas: &mut [Tarea], id: u32) -> Result<()> {
    match tareas.iter_mut().find(|t| t.id == id) {
        Some(tarea) => {
            tarea.completada = true;
            Ok(())
        }
        None => bail!("no existe una tarea con id {id}"), // anyhow::bail! = return Err(...) en una línea
    }
}

pub fn listar(tareas: &[Tarea]) -> String {
    if tareas.is_empty() {
        return "No hay tareas.".to_string();
    }
    tareas
        .iter()
        .map(|t| {
            let marca = if t.completada { "[x]" } else { "[ ]" };
            format!("{marca} #{} {}", t.id, t.descripcion)
        })
        .collect::<Vec<_>>()
        .join("\n")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn añadir_asigna_ids_incrementales() {
        let mut tareas = Vec::new();
        let id1 = añadir(&mut tareas, "primera".into());
        let id2 = añadir(&mut tareas, "segunda".into());
        assert_eq!((id1, id2), (1, 2));
    }

    #[test]
    fn completar_tarea_inexistente_falla() {
        let mut tareas = vec![Tarea::nueva(1, "única".into())];
        assert!(completar(&mut tareas, 99).is_err());
    }

    #[test]
    fn completar_marca_la_tarea_correcta() {
        let mut tareas = vec![Tarea::nueva(1, "tarea".into())];
        completar(&mut tareas, 1).unwrap();
        assert!(tareas[0].completada);
    }
}
```

Separar la lógica (`lib.rs`) de la interfaz de línea de comandos (`main.rs`, siguiente sección) es lo que permite testear `añadir`/`completar`/`listar` con `cargo test` sin tocar el sistema de ficheros ni argumentos de consola en absoluto (Tema 11.3).

## 12.5 La interfaz de línea de comandos con `clap`

```rust
// src/main.rs
use clap::{Parser, Subcommand};
use anyhow::Result;
use tareas::{almacen, añadir, completar, listar};

#[derive(Parser)]
#[command(name = "tareas", about = "Gestor de tareas simple")]
struct Cli {
    #[command(subcommand)]
    comando: Comando,
}

#[derive(Subcommand)]
enum Comando {
    /// Añade una nueva tarea
    Añadir { descripcion: String },
    /// Marca una tarea como completada
    Completar { id: u32 },
    /// Lista todas las tareas
    Listar,
}

fn main() -> Result<()> {
    let cli = Cli::parse(); // clap valida los argumentos; si son inválidos, imprime ayuda y sale
    let mut tareas = almacen::cargar()?;

    match cli.comando {
        Comando::Añadir { descripcion } => {
            let id = añadir(&mut tareas, descripcion);
            println!("Tarea #{id} añadida.");
        }
        Comando::Completar { id } => {
            completar(&mut tareas, id)?;
            println!("Tarea #{id} marcada como completada.");
        }
        Comando::Listar => {
            println!("{}", listar(&tareas));
            return Ok(()); // listar no modifica nada, no hace falta guardar
        }
    }

    almacen::guardar(&tareas)?;
    Ok(())
}
```

Nótese `fn main() -> Result<()>` (Tema 7.5): si cualquier `?` falla en toda la cadena (leer el fichero, parsear JSON, completar una tarea inexistente...), el programa termina con un mensaje de error claro y un código de salida distinto de cero — sin un solo `try/catch` ni comprobación manual de `errno`.

## 12.6 Probarlo

```bash
cargo build --release

./target/release/tareas añadir "Terminar los apuntes de Rust"
./target/release/tareas añadir "Repasar ownership"
./target/release/tareas listar
./target/release/tareas completar 1
./target/release/tareas listar
```

```
[ ] #1 Terminar los apuntes de Rust
[ ] #2 Repasar ownership
```
```
[x] #1 Terminar los apuntes de Rust
[ ] #2 Repasar ownership
```

```bash
cargo test # ejecuta los tests de src/lib.rs sin tocar tareas.json ni la consola
```

## 12.7 Qué conceptos de los temas anteriores se están usando aquí

| Tema | Dónde aparece en `tareas` |
|---|---|
| 1 — Fundamentos | Un binario único, sin runtime, candidato típico "Rust en vez de Python/Node" para CLIs |
| 3 — Tipos y control de flujo | `match` sobre `Comando`, `if`/`else` en `listar` |
| 4 — Ownership/borrowing | `&mut Vec<Tarea>` en `añadir`/`completar`, `&[Tarea]` en `listar` |
| 5 — Structs/enums/pattern matching | `struct Tarea`, `enum Comando`, desestructuración en el `match` |
| 6 — Traits | `derive(Debug, Clone, Serialize, Deserialize)`, `Parser`/`Subcommand` de `clap` |
| 7 — Manejo de errores | `anyhow::Result`, `?`, `.context(...)`, `bail!` |
| 8 — Colecciones e iteradores | `Vec<Tarea>`, `.iter().map().max()`, `.find()`, `.join()` |
| 11 — Testing y módulos | `lib.rs` separado de `main.rs`, `mod tests` con `#[cfg(test)]` |

## 12.8 De aquí en adelante

Con esta base, los siguientes pasos naturales para seguir profundizando (fuera del alcance de estos apuntes, pero como mapa de ruta) serían:
- Añadir subcomandos de borrado/edición y filtros (`listar --pendientes`).
- Sustituir el JSON plano por SQLite (`rusqlite`) para manejar más volumen de datos.
- Convertir la lógica de `lib.rs` en un servicio web con `axum` o `actix-web`, reutilizando exactamente las mismas funciones `añadir`/`completar`/`listar` (es el patrón de workspace del Tema 11.4: `core` + `cli` + `servidor`).
- Compilar a WebAssembly (`wasm-pack`) para reutilizar la misma lógica en un frontend, otro caso de uso típico de Rust mencionado en el Tema 1.7.
