# Bitácora de Requerimientos y Auditoría IA — Sesión 14
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Declaración vs Uso de Variables  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** 24531374  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tabla de símbolos mínima con declaración previa y redeclaración  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue la comparación de nombres. Al principio pensé en usar `strncmp`, pero
> con eso "x" y "xy" podrían confundirse según la longitud que le pase, y el contrato dice
> que son nombres distintos. Por eso usé `strcmp`, que compara el nombre completo hasta el
> `'\0'`. Otro detalle fue el orden de las revisiones en `declarar`: primero los argumentos
> inválidos (-1), luego si el nombre ya existe (0) y hasta el final si hay lugar. Así un
> nombre repetido devuelve 0 aunque la tabla esté llena. También cuidé que al redeclarar no
> se toque `total`, porque el caso 4 del harness comprueba que se queda en 1. En `usar` no
> hace falta tocar nada de la tabla, por eso el parámetro es `const`. Por último, limité los
> ciclos con `TABLA_MAX` para no leer fuera del arreglo si `total` viene dañado.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Gestión de memoria dinámica C99/Rust |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy en la materia Lenguajes y Autómatas II, programando en C (estándar C11).
Estoy implementando la regla de declaración previa de un compilador con una tabla de
símbolos mínima. El tipo está definido en api_sesion14.h:
  typedef struct Tabla { const char *nombres[TABLA_MAX]; int total; } Tabla;   (TABLA_MAX vale 16)
Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en tablas de
símbolos y análisis semántico.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  int declarar(Tabla *tabla, const char *nombre);
  int usar(const Tabla *tabla, const char *nombre);
Reglas: declarar devuelve 1 si insertó, 0 si el nombre ya estaba (sin insertar y sin cambiar
el total) y -1 si recibe NULL o un nombre vacío; usar devuelve 1 solo si el nombre ya fue
declarado y 0 en cualquier otro caso; la comparación es exacta, así que "x" y "xy" son
nombres distintos (no uses comparaciones parciales como strncmp).

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion14.h", sin main(), con
comentarios breves y el análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion14.h"
#include <string.h>

int declarar(Tabla *tabla, const char *nombre) {
    if (tabla == NULL || nombre == NULL || *nombre == '\0') {
        return -1;
    }
    if (usar(tabla, nombre)) {
        return 0;
    }
    if (tabla->total >= TABLA_MAX) {
        return -1;
    }
    tabla->nombres[tabla->total++] = nombre;
    return 1;
}

int usar(const Tabla *tabla, const char *nombre) {
    for (int i = 0; i < tabla->total; i++) {
        if (strcmp(tabla->nombres[i], nombre) == 0) {
            return 1;
        }
    }
    return 0;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* usar sin declarar, usar después de declarar, redeclaración, total tras redeclarar, prefijo y nombre vacío

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Memory leaks:** ninguna de las dos versiones usa malloc, la tabla vive en la pila del
harness, así que no hay nada que liberar.

**Nota sobre el crash:** el harness siempre manda una tabla válida y nombres válidos a
`usar`, por eso la versión de la IA sale en verde. Con un puntero NULL sí se cae (ver el
reporte crítico).

**Complejidad:** `usar` y `declarar` recorren la lista de nombres una vez, así que son O(n)
con n el número de símbolos. Como n nunca pasa de `TABLA_MAX` (16), en la práctica está
acotado. Cada comparación con `strcmp` cuesta O(L) con L el largo del nombre. El espacio
extra es O(1).

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **`usar` con punteros NULL:** la IA revisa los NULL en `declarar` pero no en `usar`.
>    Con `usar(NULL, "x")` se cae al leer `tabla->total`, y con `usar(&t, NULL)` se cae
>    dentro de `strcmp`. El contrato dice que `usar` devuelve 0 en cualquier caso que no
>    sea "declarado", así que debería devolver 0. Lo detecté comparando las validaciones
>    de las dos funciones. Lo solucioné revisando `tabla == NULL || nombre == NULL` al
>    inicio, como en mi versión.
> 2. **`total` dañado:** si la tabla trae `total` mayor que 16 (por ejemplo sin
>    inicializar), el ciclo lee fuera del arreglo. Yo limité los ciclos con `TABLA_MAX`.
> 3. **El -1 significa dos cosas:** `declarar` devuelve -1 tanto para argumentos
>    inválidos como para tabla llena. Quien llama no puede distinguir un error de
>    programación de un límite de espacio. El contrato no define el caso de tabla llena, así
>    que lo documento.
> 4. **Se guarda el puntero, no una copia:** si el nombre viene de un arreglo temporal
>    (por ejemplo un buffer del analizador léxico que se reutiliza), el nombre guardado
>    cambia después sin avisar. En el harness los nombres son literales y viven siempre.
> 5. **Solo detecta redeclaración en una tabla:** sin ámbitos, no distingue declarar `x`
>    dentro de un bloque y otra `x` global, como en la sesión 13.
>
> Conclusión: el harness en verde solo prueba entradas válidas. La IA resolvió bien la
> redeclaración y la comparación exacta, pero hay que auditar los punteros nulos, el
> límite de la tabla y quién es dueño de los nombres.

---

**Firma digital (GitHub):** `henari2806-ai`
