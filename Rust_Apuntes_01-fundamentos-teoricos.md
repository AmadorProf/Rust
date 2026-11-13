# Tema 1 — Fundamentos teóricos de Rust

> Tema puramente teórico. El objetivo es entender **qué problema resuelve Rust**, **qué lo hace diferente de C** y **cuándo conviene (y cuándo no conviene) elegirlo**, antes de escribir una sola línea de código.

## 1.1 ¿Qué es Rust?

Rust es un lenguaje de programación de **sistemas**, compilado, de tipado estático y fuerte, creado originalmente por Graydon Hoare en Mozilla (primera versión estable en 2015) y hoy mantenido por la Rust Foundation. Se sitúa en el mismo "nicho" que C y C++: control total sobre la memoria, sin runtime ni recolector de basura (garbage collector), rendimiento comparable al código nativo.

Su propuesta de valor se resume en una frase que repite la propia documentación oficial:

> **Rust garantiza seguridad de memoria y seguridad de concurrencia sin sacrificar rendimiento, y lo hace en tiempo de compilación, no en tiempo de ejecución.**

Esto es la clave de todo el lenguaje. En C, la seguridad de memoria depende de la disciplina del programador. En Java o Go, depende de un recolector de basura que añade overhead en tiempo de ejecución. Rust intenta la tercera vía: un **compilador que demuestra matemáticamente** (mediante un sistema de tipos y un verificador llamado *borrow checker*) que ciertas clases de errores no pueden ocurrir, y si no puede demostrarlo, el programa simplemente no compila.

## 1.2 El problema que Rust viene a resolver

Décadas de software escrito en C y C++ han demostrado un patrón: una parte muy grande de las vulnerabilidades de seguridad críticas (buffer overflows, use-after-free, double-free, data races) son errores de **gestión de memoria**, no errores de lógica de negocio. Informes de Microsoft, Google y la propia NSA/CISA han señalado reiteradamente que entre el 60% y el 70% de las vulnerabilidades graves en código C/C++ son de este tipo.

Rust nace de la observación de que estos errores son, en su mayoría, **detectables estáticamente** si el lenguaje obliga a seguir ciertas reglas sobre quién puede acceder a qué memoria y cuándo. De ahí el diseño de **ownership** (propiedad), que veremos en detalle en el Tema 4.

## 1.3 Diferencias fundamentales con C

| Aspecto | C | Rust |
|---|---|---|
| Gestión de memoria | Manual (`malloc`/`free`), sin red de seguridad | Automática vía *ownership*, determinista, sin GC |
| Punteros nulos | Permitidos, causa común de crashes | No existen; se usa `Option<T>` |
| Punteros colgantes (dangling) | Posibles y comunes | Imposibles: el *borrow checker* lo impide en compilación |
| Buffer overflow | Posible (no hay chequeo de límites por defecto) | Imposible en código seguro: los arrays/slices comprueban límites |
| Data races en concurrencia | Posibles, responsabilidad del programador | Imposibles en código seguro: el sistema de tipos lo impide (`Send`/`Sync`) |
| Manejo de errores | Códigos de retorno / `errno`, fácil de ignorar | `Result<T, E>` y `Option<T>`, el compilador obliga a gestionarlos |
| Null-terminated strings | Sí (`char*`), fuente de errores (`strlen`, overflows) | `String`/`&str` con longitud explícita y validación UTF-8 |
| Macros | Preprocesador textual, sin higiene | Macros higiénicas (`macro_rules!`) o procedurales, con AST real |
| Gestión de dependencias | Inexistente en el lenguaje (make, autotools, vendoring manual) | `cargo` integrado: build, tests, dependencias, docs, publicación |
| Genéricos | No existen (se simulan con macros o `void*`) | Genéricos reales con monomorfización, como C++ templates |
| Herencia / polimorfismo | No hay (se simula con structs + punteros a función) | Traits (interfaces) + *trait objects* para polimorfismo dinámico |
| Overhead del "modelo seguro" | N/A | Cero en tiempo de ejecución: las comprobaciones son en compilación |

### Ejemplo conceptual: use-after-free

En C, este patrón compila y es un bug silencioso hasta que explota en producción:

```c
char *get_name() {
    char buffer[50] = "Hola";
    return buffer; // devuelve un puntero a memoria de la pila ya liberada
}
```

En Rust, el equivalente **no compila**. El compilador analiza el *lifetime* (tiempo de vida) de `buffer` y ve que no puede sobrevivir a la función, así que rechaza el código con un error explicando exactamente por qué. El error se detecta en el ordenador del desarrollador, no en el servidor de un cliente.

### Ejemplo conceptual: data race

En C, dos hilos escribiendo en la misma variable sin sincronización compilan sin ningún aviso; el comportamiento resultante es indefinido y puede funcionar "bien" durante meses hasta fallar de forma intermitente y muy difícil de depurar. En Rust, intentar compartir una variable mutable entre hilos sin un mecanismo de sincronización (`Mutex`, `Arc`, canales, etc.) es un **error de compilación**, no un bug en producción.

## 1.4 El "costo cero de abstracción" (zero-cost abstractions)

Una idea central, heredada de C++ pero llevada más lejos, es que las abstracciones de alto nivel (iteradores, genéricos, closures, traits) deben compilar a código tan eficiente como si se hubiera escrito a mano en bajo nivel. Ejemplo: iterar con `.map().filter().sum()` sobre un vector genera, tras optimización, código equivalente a un bucle `for` manual con acceso directo a memoria — sin asignaciones intermedias, sin despacho dinámico salvo que se pida explícitamente.

Esto es importante porque significa que elegir Rust no es un compromiso entre "código bonito" y "código rápido": se puede escribir código expresivo y de alto nivel sin pagar penalización de rendimiento, algo que no es cierto en Python, Java o incluso C++ en muchos casos (p. ej. `std::function` o excepciones sí tienen coste).

## 1.5 Ownership, Borrowing y Lifetimes (preview conceptual)

Estos tres conceptos son el núcleo diferencial de Rust y se desarrollan en profundidad en el Tema 4, pero conviene adelantar la idea:

- **Ownership (propiedad)**: cada valor tiene exactamente un "dueño" en cada momento. Cuando el dueño sale de ámbito (*scope*), el valor se libera automáticamente — de forma determinista, sin GC, en tiempo de compilación se inserta la llamada equivalente a `free`/destructor.
- **Borrowing (préstamo)**: en vez de copiar o transferir la propiedad constantemente, se puede "prestar" una referencia a un valor (`&T` inmutable o `&mut T` mutable), con la regla de que **o bien hay múltiples referencias inmutables, o bien una sola referencia mutable, nunca ambas a la vez**. Esta regla, verificada en compilación, es lo que elimina los data races.
- **Lifetimes (tiempos de vida)**: el compilador rastrea durante cuánto tiempo es válida cada referencia para garantizar que nunca se usa una referencia a datos que ya no existen (dangling references).

Esto es, en esencia, una versión formalizada y verificada automáticamente de las reglas que un programador de C experimentado ya sigue "en su cabeza" (no liberar dos veces, no usar un puntero después de liberarlo, no tener dos punteros mutables al mismo dato). Rust convierte esa disciplina informal en reglas del lenguaje comprobadas por el compilador.

## 1.6 Motor del lenguaje: compilador, LLVM y el ecosistema

- **rustc**: el compilador de Rust, escrito en el propio Rust (es *self-hosted*). Usa **LLVM** como backend de generación de código (igual que Clang para C/C++), lo que permite generar código máquina muy optimizado para una enorme variedad de arquitecturas.
- **Cargo**: el gestor de paquetes y build system oficial, integrado desde el primer día (a diferencia de C, donde no hay un estándar: cada proyecto usa Make, CMake, Meson, Bazel...). Cargo gestiona dependencias (`crates.io`), compila, ejecuta tests, genera documentación y publica paquetes.
- **El borrow checker**: la parte del compilador (actualmente evolucionando hacia "Polonius", una reimplementación más precisa) que analiza el grafo de referencias y lifetimes del programa para garantizar las reglas de ownership.
- **No hay runtime obligatorio ni GC**: el binario resultante es comparable en tamaño y comportamiento a uno de C. Existe un *runtime* mínimo (para cosas como el manejo de pánico/unwinding), pero no hay un recolector de basura ni una máquina virtual.
- **`unsafe`**: una válvula de escape explícita. Rust permite marcar bloques de código como `unsafe` para hacer operaciones que el compilador no puede verificar (desreferenciar punteros crudos, FFI con C, etc.). Esto es clave: Rust no prohíbe el bajo nivel, lo **aísla y lo hace explícito**, de modo que si hay un bug de memoria, se sabe que solo puede estar en las zonas marcadas `unsafe` (normalmente una fracción mínima del código).

## 1.7 ¿Cuándo conviene usar Rust?

Rust brilla especialmente cuando se dan una o varias de estas condiciones:

- **Rendimiento crítico + necesidad de seguridad**: sistemas donde un fallo de memoria es inaceptable (navegadores, motores de bases de datos, infraestructura de red, criptografía, sistemas embebidos críticos). Ejemplos reales: Firefox (componentes de Servo/Stylo), el kernel de Linux (soporte experimental para drivers en Rust), Discord (reescribió partes de su backend de Go a Rust por latencia y uso de memoria), AWS (Firecracker, componentes de S3), Dropbox (motor de sincronización de archivos), Cloudflare (proxy y workers).
- **Concurrencia/paralelismo intensivo**: cuando se necesita exprimir varios núcleos sin arriesgarse a data races. El sistema de tipos hace que el paralelismo "a prueba de fallos" sea mucho más accesible que en C/C++.
- **Sustituir C/C++ en código nuevo o crítico**: cuando se está escribiendo algo nuevo que tradicionalmente se habría hecho en C/C++ (parsers, motores de juegos, herramientas CLI de alto rendimiento, compiladores, sistemas operativos, firmware), Rust da las mismas capacidades de bajo nivel con muchísimas menos probabilidades de bugs de memoria.
- **WebAssembly**: Rust tiene uno de los mejores soportes de compilación a WASM del ecosistema, con binarios pequeños y sin runtime pesado — ideal para lógica compartida entre frontend y backend, o plugins sandboxeados.
- **Herramientas de línea de comandos**: el ecosistema (`clap`, `serde`, etc.) y el hecho de producir un único binario estático, rápido de arrancar y sin dependencias de runtime, hace de Rust una opción excelente frente a Python o Node para CLIs que se distribuyen a terceros (ej: `ripgrep`, `fd`, `bat`, `uv`, `exa`).
- **Equipos grandes / código de larga vida**: el compilador actúa como un revisor de código implacable. Esto reduce (no elimina) la necesidad de tests exhaustivos para cierta clase de bugs, y hace que refactorizar código antiguo sea mucho menos arriesgado que en C/C++ ("si compila, probablemente no tiene use-after-free ni data races").

## 1.8 ¿Cuándo NO conviene (o no es la mejor opción)?

Ser honesto sobre las desventajas es tan importante como conocer las ventajas:

- **Prototipado muy rápido / scripts descartables**: el *borrow checker* impone una curva de aprendizaje y una "fricción" inicial (el famoso "fighting the borrow checker") que puede ralentizar iterar una idea rápida. Python, Go o incluso JavaScript son mejores para explorar ideas desechables.
- **Equipos sin experiencia y plazos muy ajustados**: la curva de aprendizaje de ownership/lifetimes/traits es real, sobre todo si el equipo viene de lenguajes con GC (Python, Java, JS) y nunca ha pensado en quién "posee" la memoria. El tiempo de onboarding debe presupuestarse.
- **Tiempos de compilación**: `rustc` + el borrow checker + la monomorfización de genéricos hacen que compilar proyectos grandes en Rust sea notablemente más lento que en Go, y a veces más lento que C++ (aunque mejor que compilar C++ con templates pesados tipo Boost). Para ciclos de feedback ultrarrápidos esto puede pesar.
- **Cuando ya existe un ecosistema maduro en otro lenguaje que cubre exactamente la necesidad**: si el cuello de botella no es rendimiento ni seguridad de memoria (p. ej. un CRUD típico con bajo tráfico), introducir Rust puede ser sobreingeniería: se paga la fricción del lenguaje sin necesitar sus garantías.
- **Interfaces gráficas de escritorio tradicionales**: el ecosistema GUI de Rust (egui, iced, Tauri...) es funcional pero mucho menos maduro que Qt/C++, Electron/JS o SwiftUI. Para ese caso de uso concreto, hoy por hoy, puede no ser la opción más productiva.
- **Cuando el GC no es un problema real**: si la aplicación no tiene requisitos de latencia estrictos (pausas de GC aceptables) y el equipo ya domina Java/Go/C#, el coste de migrar a un lenguaje sin GC rara vez se justifica solo por "rendimiento en abstracto".

## 1.9 Rust frente a otros lenguajes de systems programming (resumen)

- **Rust vs C**: mismo nivel de control, pero Rust añade seguridad de memoria verificada y gestión de dependencias moderna a cambio de una curva de aprendizaje mayor y compilación más lenta.
- **Rust vs C++**: ambos con zero-cost abstractions y sin GC; Rust tiene un sistema de ownership obligatorio y verificado (C++ tiene `unique_ptr`/`shared_ptr` pero son opcionales y no impiden errores por diseño), y un ecosistema de paquetes moderno (Cargo) frente a la fragmentación histórica de C++ (aunque C++ tiene 40 años más de librerías y bases de código existentes).
- **Rust vs Go**: Go es más simple, compila casi instantáneamente y tiene GC; es excelente para servicios de red con concurrencia moderada. Rust no tiene GC, da más control de memoria y rendimiento, pero con mayor complejidad. Go gana en velocidad de desarrollo; Rust gana en rendimiento predecible y ausencia de pausas de GC.
- **Rust vs Python/JS**: no son comparables directamente (lenguajes interpretados/alto nivel vs sistemas), pero Rust se usa cada vez más para escribir las partes críticas de rendimiento de librerías Python/Node (ej: `ruff`, `polars`, `pydantic-core`, partes de Node.js vía `napi-rs`), combinando la ergonomía de Python/JS en la capa de aplicación con Rust en el núcleo de cómputo.

## 1.10 Resumen del tema

- Rust resuelve el problema histórico de los bugs de memoria y concurrencia en lenguajes de sistemas, moviendo la verificación de tiempo de ejecución (o directamente la ausencia de verificación, como en C) a tiempo de compilación.
- Lo logra sin recolector de basura y sin coste de rendimiento en tiempo de ejecución, gracias al modelo de ownership/borrowing/lifetimes y a las "zero-cost abstractions".
- Es la mejor opción cuando se necesita rendimiento nativo **y** seguridad/concurrencia robusta a la vez; es una opción discutible cuando se necesita iterar muy rápido, el equipo no tiene tiempo de aprender la curva, o el problema no tiene requisitos reales de rendimiento/memoria.
- A partir de aquí, el resto de temas son prácticos: instalaremos el toolchain y escribiremos código para ver estos conceptos (ownership, traits, errores, concurrencia...) en acción.

**Siguiente paso →** [Tema 2: Entorno y Cargo](02-entorno-y-cargo.md)
