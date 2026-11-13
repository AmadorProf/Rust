# Tema 10 — Smart pointers y gestión de memoria avanzada

## 10.1 ¿Por qué hacen falta "punteros inteligentes" si ya hay ownership?

El modelo de ownership simple del Tema 4 (un dueño, transferencia por `move`, liberación al salir de scope) cubre la mayoría de los casos, pero no todos:

- ¿Qué pasa si un valor necesita vivir en el **heap** explícitamente (p. ej. porque es muy grande, o porque su tamaño no se conoce en compilación, como en tipos recursivos)?
- ¿Qué pasa si **varias partes del programa** necesitan ser, legítimamente, "dueñas" del mismo dato (un grafo, un árbol con referencias cruzadas)?
- ¿Qué pasa si se necesita mutar algo a través de una referencia compartida, rompiendo la regla estricta del Tema 4, pero de forma controlada?

Para estos casos, Rust ofrece **smart pointers**: tipos que se comportan como punteros (implementan `Deref`) pero añaden semántica extra. Es el mismo espíritu que `unique_ptr`/`shared_ptr` de C++, pero integrado con el borrow checker.

## 10.2 `Box<T>`: un valor en el heap con un solo dueño

```rust
fn main() {
    let b = Box::new(5); // 5 vive en el heap, b es su único dueño
    println!("{b}");
} // al salir de scope, se libera automáticamente, como cualquier otro valor
```

Casos de uso típicos:

- **Tipos recursivos**, cuyo tamaño sería infinito sin indirección:

```rust
enum Lista {
    Nodo(i32, Box<Lista>), // Box rompe la recursión infinita de tamaño
    Fin,
}

use Lista::{Nodo, Fin};

fn main() {
    let lista = Nodo(1, Box::new(Nodo(2, Box::new(Nodo(3, Box::new(Fin))))));
    let _ = lista;
}
```

- **Trait objects** (`Box<dyn Trait>`, visto en el Tema 6).
- Mover valores grandes sin copiarlos en la pila repetidamente.

`Box<T>` no tiene coste de runtime más allá de la indirección de puntero: es, conceptualmente, `malloc` + liberación automática garantizada, sin conteo de referencias ni overhead adicional.

## 10.3 `Rc<T>`: ownership compartido (un solo hilo)

Cuando varias partes del código necesitan ser dueñas del mismo dato simultáneamente (p. ej. un nodo de un grafo referenciado desde varios sitios), `Rc<T>` ("**R**eference **C**ounted") permite tener múltiples dueños, liberando el valor solo cuando el último `Rc` se destruye:

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("dato compartido"));
    println!("referencias: {}", Rc::strong_count(&a)); // 1

    let b = Rc::clone(&a); // NO clona el String, solo incrementa el contador
    println!("referencias: {}", Rc::strong_count(&a)); // 2

    {
        let c = Rc::clone(&a);
        println!("referencias: {}", Rc::strong_count(&a)); // 3
        let _ = c;
    } // c se destruye, contador vuelve a 2

    println!("referencias: {}", Rc::strong_count(&a)); // 2
    println!("{a} {b}");
}
```

`Rc<T>` **no es seguro entre hilos** (su contador no es atómico, por rendimiento): el compilador lo impide directamente si se intenta enviar a otro hilo (recuerda `Send`/`Sync` del Tema 9). Para el equivalente multi-hilo se usa `Arc<T>`.

## 10.4 `RefCell<T>`: mutabilidad controlada en tiempo de ejecución

La regla del Tema 4 ("una `&mut` o varias `&`, nunca ambas") se verifica **en compilación**. Hay casos legítimos donde esa verificación estática es demasiado estricta para el patrón que se necesita (p. ej. estructuras de datos con referencias internas compartidas). `RefCell<T>` mueve esa misma regla **a tiempo de ejecución**: permite mutar un valor incluso a través de una referencia compartida (`&RefCell<T>`), pero sigue comprobando las reglas de préstamo — si se violan, hace `panic!` en vez de fallar en compilación.

```rust
use std::cell::RefCell;

fn main() {
    let dato = RefCell::new(5);

    {
        let mut valor = dato.borrow_mut(); // préstamo mutable, verificado en runtime
        *valor += 1;
    } // se libera el préstamo aquí

    println!("{}", dato.borrow());

    // let _a = dato.borrow_mut();
    // let _b = dato.borrow_mut(); // PANIC en runtime: "already borrowed: BorrowMutError"
}
```

**Importante**: esto no es "menos seguro" que el borrow checker estático — sigue siendo imposible tener dos referencias mutables simultáneas en código seguro. Solo cambia *cuándo* se detecta la violación (ejecución en vez de compilación), a cambio de más flexibilidad. El patrón combinado `Rc<RefCell<T>>` es muy común para estructuras con ownership compartido y mutación controlada en programas de un solo hilo (p. ej. árboles/grafos con referencias padre-hijo).

## 10.5 Tabla resumen

| Tipo | Ownership | Mutabilidad | Multi-hilo | Verificación |
|---|---|---|---|---|
| `Box<T>` | Único dueño | Según `&`/`&mut` normal | Sí (si `T: Send`) | Compilación |
| `Rc<T>` | Compartido (contador no atómico) | Solo inmutable por defecto | No | Compilación |
| `Arc<T>` | Compartido (contador atómico) | Solo inmutable por defecto | Sí | Compilación |
| `RefCell<T>` | Normal (no es sobre ownership) | Mutable incluso vía `&` | No | **Runtime** (panic si se viola) |
| `Mutex<T>` / `RwLock<T>` | Normal | Mutable vía lock | Sí | Runtime (bloqueo, no panic) |

## 10.6 `unsafe`: la válvula de escape

Todo lo anterior es código **seguro** (*safe Rust*). Existen operaciones que el compilador no puede verificar automáticamente pero que a veces son necesarias: desreferenciar punteros crudos, llamar a funciones de C vía FFI, implementar estructuras de datos de muy bajo nivel (como hace la propia librería estándar internamente para `Vec` o `Rc`). Para esto existe la palabra clave `unsafe`:

```rust
fn main() {
    let mut num = 5;
    let r1 = &num as *const i32;       // puntero crudo inmutable
    let r2 = &mut num as *mut i32;     // puntero crudo mutable

    unsafe {
        println!("r1: {}", *r1); // desreferenciar un puntero crudo requiere unsafe
        *r2 = 10;
    }
}
```

`unsafe` **no desactiva el borrow checker en todo el programa**: solo habilita, dentro de ese bloque, un puñado de operaciones concretas (desreferenciar punteros crudos, llamar funciones `unsafe`, acceder a `static mut`, implementar traits `unsafe` como `Send`/`Sync` manualmente, acceder a campos de uniones). El resto de las reglas del lenguaje siguen aplicando igual.

Esto conecta directamente con el Tema 1.6: Rust no prohíbe el bajo nivel, lo **aísla explícitamente**. En un proyecto grande, si aparece un bug de memoria, la primera pregunta de depuración es "¿dónde hay `unsafe`" — que suele ser una fracción diminuta del código (a menudo 0%, en aplicaciones típicas que no hacen FFI ni estructuras de datos a medida).

## 10.7 Ejercicio

1. Implementa un árbol binario simple usando `Rc<RefCell<Nodo>>`, donde cada `Nodo` tiene un valor `i32` y dos hijos opcionales (`Option<Rc<RefCell<Nodo>>>`).
2. Escribe una función que inserte valores en el árbol (orden simple, sin balanceo) y otra que lo recorra in-order imprimiendo los valores.
3. (Extra) Investiga por qué este patrón (`Rc<RefCell<T>>`) puede provocar **memory leaks** (no "unsafe", pero sí fugas) si se crean referencias cíclicas, y qué tipo de la librería estándar (`Weak<T>`) existe para romper esos ciclos.

**Siguiente →** [Tema 11: Testing, módulos y organización de proyectos](11-testing-modulos-organizacion.md)
