# Bitácora de Requerimientos y Auditoría IA — Sesión 16
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Firmas de Función y Parámetros Formales  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** 24531374  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Verificador de llamadas con registro de firmas y aridad  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue separar los tres resultados posibles, porque 0 y -1 se parecen pero
> significan cosas distintas: 0 es "la función existe pero la llamaste con el número
> equivocado de argumentos" y -1 es "esa función ni existe". Por eso primero busco el
> nombre, y solo cuando lo encuentro comparo la aridad. Si comparara la aridad antes, una
> función inexistente se confundiría con una de aridad incorrecta. También me fijé en que
> debo comparar la aridad de ESA firma (`catalogo[i].aridad`) contra `nargs`, y no contra
> el total de firmas del catálogo, que era un error fácil de cometer porque ambos son
> enteros. Para `main` con aridad 0, la comparación `0 == 0` ya da 1, y con 2 argumentos
> da 0, así que no hizo falta un caso especial. Por último, revisé que ningún nombre del
> catálogo sea NULL antes de pasarlo a `strcmp`, porque eso tronaría el programa.

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
Estoy implementando la verificación de llamadas a función de un compilador. El tipo está
definido en api_sesion16.h:
  typedef struct Funcion { const char *nombre; int aridad; } Funcion;
Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en análisis
semántico y verificación de llamadas.

ESPECIFICACIÓN: Implementa exactamente esta firma, sin cambiarla:
  int verificar_llamada(const Funcion *catalogo, int total, const char *nombre, int nargs);
Reglas: busca el nombre de forma exacta recorriendo el catálogo completo; devuelve 1 si la
función existe y su aridad es igual a nargs, 0 si existe pero la aridad no coincide, y -1 si
el nombre no existe; una función con aridad 0 solo acepta nargs igual a 0; un catálogo NULL,
un nombre NULL o una aridad de llamada negativa devuelven -1. Compara la aridad declarada
con nargs, nunca con el número de firmas del catálogo.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion16.h", sin main(), con
comentarios breves y el análisis de complejidad de la función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion16.h"
#include <string.h>

int verificar_llamada(const Funcion *catalogo, int total, const char *nombre, int nargs) {
    if (catalogo == NULL || nombre == NULL) {
        return -1;
    }
    for (int i = 0; i < total; i++) {
        if (strcmp(catalogo[i].nombre, nombre) == 0) {
            return catalogo[i].aridad == nargs;
        }
    }
    return -1;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* suma con 3, 2 y 4 argumentos, función no registrada, main sin argumentos y nombre NULL

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Memory leaks:** ninguna de las dos versiones usa malloc, el catálogo vive en la pila del
harness, así que no hay nada que liberar.

**Nota sobre el crash:** el harness solo manda catálogos con nombres válidos y nunca una
aridad negativa en una función que sí existe, por eso la versión de la IA sale en verde.
Con esos casos se equivoca o se cae (ver el reporte crítico).

**Complejidad:** `verificar_llamada` recorre el catálogo una vez, así que es O(n) con n el
número de firmas, y cada comparación con `strcmp` cuesta O(L) con L el largo del nombre.
El espacio extra es O(1).

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **Aridad negativa en una función que sí existe:** el contrato dice que una aridad
>    negativa devuelve -1. La IA solo revisa catálogo y nombre NULL, así que con
>    `verificar_llamada(c, 1, "suma", -1)` devuelve 0 (aridad incorrecta) en lugar de -1
>    (argumentos inválidos). El harness solo prueba el nombre NULL, por eso no lo notó.
>    Lo detecté releyendo las reglas del contrato contra cada `if` del código. Lo
>    solucioné agregando `nargs < 0` a la validación inicial, como en mi versión.
> 2. **Nombre NULL dentro del catálogo:** si una firma tiene `nombre == NULL`, la IA lo
>    pasa a `strcmp` y el programa se cae. Yo salto esas firmas con `continue`.
> 3. **`total` negativo:** la IA no lo valida. No se cae, porque el ciclo no se ejecuta y
>    devuelve -1, pero lo hace por casualidad y no por una revisión explícita.
> 4. **Nombres repetidos:** si el catálogo tiene dos firmas con el mismo nombre y distinta
>    aridad (como una sobrecarga), las dos versiones se quedan con la primera y dan 0 para
>    la otra aridad, aunque la segunda sí sea válida. El contrato no define este caso, así
>    que lo documento.
> 5. **Solo se verifica la cantidad, no los tipos:** `suma(1, 2, 3)` y `suma("a", 2, 3)`
>    pasan igual. Un compilador real también compara el tipo de cada parámetro. Queda fuera
>    del contrato de esta sesión.
>
> Conclusión: el harness en verde solo prueba 6 casos con datos bien formados. La IA
> resolvió bien la búsqueda y la comparación de aridad, pero hay que releer el contrato
> completo y auditar cada regla que el harness no cubre.

---

**Firma digital (GitHub):** `henari2806-ai`
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> 

---
**Firma digital:** [Usuario de GitHub]
