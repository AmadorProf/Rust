# Tema 9 — Concurrencia

> Este es uno de los temas donde la inversión hecha en los Temas 4 (ownership/borrowing) da más rendimiento: Rust convierte los data races, uno de los bugs más difíciles de depurar en C/C++, en **errores de compilación**.

## 9.1 Hilos nativos (`std::thread`)

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("hilo secundario: {i}");
            thread::sleep(Duration::from_millis(50));
        }
    });

    for i in 1..3 {
        println!("hilo principal: {i}");
        thread::sleep(Duration::from_millis(50));
    }

    handle.join().unwrap(); // espera a que el hilo secundario termine
}
```

Esto es directamente comparable a `pthread_create`/`pthread_join` en C, pero sin gestión manual de punteros a datos compartidos entre hilos.

### `move` closures: transferir ownership al hilo

```rust
fn main() {
    let datos = vec![1, 2, 3];

    let handle = thread::spawn(move || { // move: el closure toma ownership de `datos`
        println!("{datos:?}");
    });
    // println!("{datos:?}"); // ERROR: datos fue movido al hilo, ya no es válido aquí

    handle.join().unwrap();
}
```

El compilador obliga a decidir explícitamente: o el hilo **toma ownership** de los datos (`move`), o se comparten mediante un mecanismo seguro (ver 9.2/9.3). No hay forma de que dos hilos tengan acceso mutable simultáneo "por accidente" — el intento ni siquiera compila.

## 9.2 `Mutex<T>` y `Arc<T>`: compartir estado entre hilos de forma segura

En C, compartir una variable entre hilos requiere disciplina manual con `pthread_mutex_t`, y nada impide olvidar bloquear el mutex antes de acceder al dato — es responsabilidad 100% del programador, y el compilador no puede ayudar.

En Rust, un `Mutex<T>` **envuelve el dato**, de modo que es físicamente imposible acceder a él sin pasar por `.lock()`:

```rust
use std::sync::Mutex;

fn main() {
    let contador = Mutex::new(0);

    {
        let mut num = contador.lock().unwrap(); // bloquea, devuelve un "guard"
        *num += 1;
    } // el lock se libera automáticamente aquí (RAII, igual que cualquier drop)

    println!("{:?}", contador.lock().unwrap());
}
```

Para compartir ese `Mutex` entre varios hilos, se necesita `Arc<T>` ("**A**tomically **R**eference **C**ounted"), el equivalente thread-safe de `Rc<T>` (conteo de referencias de un solo hilo, Tema 10):

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let contador = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let contador = Arc::clone(&contador); // clona el puntero (incrementa el contador de referencias), no el dato
        let handle = thread::spawn(move || {
            let mut num = contador.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Resultado: {}", *contador.lock().unwrap()); // 10, garantizado, sin data race
}
```

Si se intentara compartir el `Mutex` directamente sin `Arc` (p. ej. moviéndolo a dos hilos a la vez), **el código no compila**: el compilador detecta que un mismo valor no puede tener dos dueños simultáneos (Tema 4), y Rust exige `Arc` precisamente para expresar "propiedad compartida con conteo de referencias atómico, seguro entre hilos".

## 9.3 `Send` y `Sync`: las reglas detrás de todo esto

Dos traits (normalmente invisibles, implementados automáticamente por el compilador cuando corresponde) formalizan qué se puede compartir entre hilos:

- **`Send`**: un tipo es `Send` si es seguro **transferir su ownership** a otro hilo.
- **`Sync`**: un tipo es `Sync` si es seguro **acceder a él por referencia desde varios hilos a la vez**.

La inmensa mayoría de tipos son ambos automáticamente. Los que no lo son (p. ej. `Rc<T>`, que no usa conteo atómico) generan un error de compilación si se intenta cruzar un límite de hilo con ellos — ese es, literalmente, el mecanismo que previene los data races: el compilador rechaza el programa antes de que exista el bug.

## 9.4 Canales (`mpsc`): paso de mensajes

Alternativa al estado compartido: comunicar hilos enviándose mensajes, como en Go (`channels`) o Erlang.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel(); // tx: transmisor, rx: receptor

    thread::spawn(move || {
        let mensajes = vec!["hola", "desde", "el hilo"];
        for m in mensajes {
            tx.send(m.to_string()).unwrap();
        }
    });

    for recibido in rx { // itera hasta que todos los tx se cierran
        println!("Recibido: {recibido}");
    }
}
```

## 9.5 Concurrencia asíncrona: `async`/`await`

Para I/O intensivo (servidores web, miles de conexiones concurrentes) los hilos del sistema operativo son demasiado pesados (cada uno reserva stack, cambios de contexto caros). Rust ofrece `async`/`await` en el lenguaje, pero **sin runtime incluido** — a diferencia de Go o Node, hay que elegir un *executor* (el más usado con diferencia es **Tokio**):

```rust
// Cargo.toml: tokio = { version = "1", features = ["full"] }

async fn obtener_datos(id: u32) -> String {
    // simula una operación de I/O (red, disco...) sin bloquear el hilo
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    format!("datos del id {id}")
}

#[tokio::main]
async fn main() {
    // ejecutar varias tareas asíncronas CONCURRENTEMENTE en el mismo hilo (o pocos hilos)
    let (r1, r2, r3) = tokio::join!(
        obtener_datos(1),
        obtener_datos(2),
        obtener_datos(3),
    );

    println!("{r1}, {r2}, {r3}");
}
```

Una función `async fn` no ejecuta nada al llamarla: devuelve un `Future` (un valor que representa "un cómputo que eventualmente dará un resultado"), que no avanza hasta que algo lo `.await`-ea dentro de un runtime como Tokio. Esto permite manejar decenas de miles de conexiones concurrentes con un puñado de hilos del sistema operativo, de forma similar a `epoll`/`io_uring` en C pero con una sintaxis secuencial legible en vez de callbacks o máquinas de estados manuales.

## 9.6 Hilos vs async: ¿cuándo cada uno?

| | Hilos (`std::thread`) | Async (`tokio`, etc.) |
|---|---|---|
| Mejor para | Trabajo con mucha CPU (cómputo paralelo) | Trabajo con mucha I/O (red, disco, miles de conexiones) |
| Coste por unidad | Alto (stack de varios KB/MB, cambio de contexto del SO) | Bajo (una tarea async puede pesar bytes) |
| Complejidad | Menor, más directo | Mayor (ecosistema de runtimes, `Pin`, `Send` en Futures...) |
| Dependencias | Ninguna, es `std` | Requiere un runtime externo (Tokio, async-std...) |

## 9.7 Resumen frente a C

En C, escribir código concurrente correcto exige disciplina manual total: mutexes de pthreads, cuidado extremo con qué se comparte y cómo, y herramientas externas (ThreadSanitizer, Valgrind/Helgrind) para *intentar* detectar data races en tiempo de ejecución — y aun así, solo detectan los que ocurren durante la ejecución concreta que se está probando. En Rust, la clase entera de "data race por olvidar sincronizar un acceso compartido mutable" se detecta **siempre**, en todo camino de ejecución posible, en el momento de compilar — no en un test concreto que pueda no disparar la condición de carrera.

## 9.8 Ejercicio

1. Lanza 5 hilos que cada uno incremente un contador compartido (`Arc<Mutex<i32>>`) 1000 veces, y comprueba que el resultado final es exactamente 5000.
2. Reescribe el ejercicio 1 usando `std::sync::atomic::AtomicI32` en vez de `Mutex<i32>` (más eficiente para un contador simple, investiga `fetch_add`).
3. (Extra, requiere Tokio) Escribe una función `async fn` que simule descargar 3 "archivos" (con `tokio::time::sleep`) y mide con `std::time::Instant` cuánto tarda hacerlo con `tokio::join!` (concurrente) frente a hacerlo con tres `.await` secuenciales.

**Siguiente →** [Tema 10: Smart pointers y gestión de memoria avanzada](10-smart-pointers-memoria.md)
