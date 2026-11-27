# Bitácora de Requerimientos y Auditoría IA — Sesión 13
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Manejo de Ámbitos (Scope Global vs Local)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** 24531374  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Pila de ámbitos enlazados y búsqueda jerárquica  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue entender que `declarar` y `buscar` trabajan distinto. `declarar` solo
> mira el ámbito actual: si también revisara los padres, no se podría tapar una variable
> global con una local del mismo nombre (el shadowing). En cambio `buscar` empieza en el
> ámbito actual y va subiendo por `padre` hasta la raíz, y se detiene en el primero que
> encuentra, por eso la local le gana a la global. Otro detalle fue lo que devuelve
> `buscar`: no es el símbolo sino el nombre del ámbito donde se encontró, como "global" o
> "bloque". También tuve que acordarme de revisar si el nombre ya existe ANTES de revisar
> si el ámbito está lleno, porque un nombre repetido debe devolver 0 aunque no quepa nada
> más. Por último, el arreglo `nombres` solo tiene 8 lugares, así que limité los ciclos
> con `AMBITO_MAX` para no leer fuera del arreglo si `total` viene dañado.

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
Estoy implementando los ámbitos (scope) de un compilador como una pila de ámbitos
enlazados. El tipo está definido en api_sesion13.h:
  typedef struct Ambito { const char *nombre; const char *nombres[AMBITO_MAX]; int total;
                          struct Ambito *padre; } Ambito;   (AMBITO_MAX vale 8)
Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en tablas de
símbolos y resolución de nombres.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  int declarar(Ambito *ambito, const char *nombre);
  const char *buscar(const Ambito *ambito, const char *nombre);
Reglas: declarar guarda el nombre solo en el ámbito actual y no consulta los padres (el
mismo nombre puede existir una vez en cada nivel); devuelve 1 si insertó, 0 si el nombre ya
estaba en ese ámbito, y -1 si el ámbito está lleno o recibe NULL o un nombre vacío; buscar
empieza en el ámbito actual y sube por padre hasta la raíz, y devuelve el nombre del ámbito
más cercano donde encontró el identificador (no el símbolo); si no existe devuelve NULL sin
modificar nada.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion13.h", sin main(), con
comentarios breves y el análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion13.h"
#include <string.h>

int declarar(Ambito *ambito, const char *nombre) {
    if (ambito == NULL || nombre == NULL || *nombre == '\0') {
        return -1;
    }
    for (int i = 0; i < ambito->total; i++) {
        if (strcmp(ambito->nombres[i], nombre) == 0) {
            return 0;
        }
    }
    if (ambito->total >= AMBITO_MAX) {
        return -1;
    }
    ambito->nombres[ambito->total++] = nombre;
    return 1;
}

const char *buscar(const Ambito *ambito, const char *nombre) {
    for (const Ambito *a = ambito; a != NULL; a = a->padre) {
        for (int i = 0; i < a->total; i++) {
            if (strcmp(a->nombres[i], nombre) == 0) {
                return a->nombre;
            }
        }
    }
    return NULL;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* declarar, búsqueda en el mismo ámbito, subir al padre, shadowing, tres niveles hasta la raíz y nombre no declarado

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Memory leaks:** ninguna de las dos versiones usa malloc, los ámbitos viven en la pila del
harness, así que no hay nada que liberar.

**Nota sobre el crash:** el harness siempre manda nombres válidos y ámbitos bien armados,
por eso la versión de la IA sale en verde. Con un nombre NULL en `buscar` sí se cae (ver
el reporte crítico).

**Complejidad:** `declarar` recorre como máximo 8 nombres, así que es O(1) por estar
acotado por `AMBITO_MAX`. `buscar` es O(d · k), con d la profundidad de la cadena de
ámbitos y k ≤ 8 los nombres por ámbito. El espacio extra es O(1) porque el recorrido es
un ciclo y no una recursión.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **`buscar` con nombre NULL:** la IA no revisa `nombre` y se lo pasa a `strcmp`, así
>    que `buscar(&g, NULL)` se cae. Lo detecté comparando las validaciones de `declarar`
>    (que sí revisa NULL) con las de `buscar` (que no revisa nada). Lo solucioné
>    devolviendo NULL al inicio, como en mi versión.
> 2. **`total` dañado:** si un ámbito trae `total` mayor que 8 (por ejemplo por no haber
>    sido inicializado), el ciclo de la IA lee fuera del arreglo `nombres`. Yo limité los
>    ciclos con `i < AMBITO_MAX`.
> 3. **Ciclo en la cadena de padres:** si por error un ámbito es padre de sí mismo o hay
>    un círculo, `buscar` entra en un ciclo infinito cuando el nombre no existe. En un
>    compilador real un error así dejaría la compilación colgada.
> 4. **Se guarda el puntero, no una copia:** `declarar` guarda el mismo `const char *` que
>    recibe. Si quien llama usa un arreglo temporal que luego se reutiliza o se libera, el
>    ámbito queda con un puntero colgante o con el nombre cambiado. En el harness los
>    nombres son literales y viven siempre, por eso no pasa nada.
> 5. **Ámbito sin nombre:** `buscar` devuelve `a->nombre`. Si ese campo es NULL, el
>    resultado es NULL aunque el identificador sí se haya encontrado, y quien llama lo
>    confunde con "no existe". Lo documento, el contrato no lo cubre.
>
> Conclusión: el harness en verde solo prueba ámbitos bien formados y nombres válidos. La
> IA resolvió bien el ocultamiento (shadowing) y la subida por los padres, pero hay que
> auditar los punteros nulos y los datos dañados.

---

**Firma digital (GitHub):** `henari2806-ai`
