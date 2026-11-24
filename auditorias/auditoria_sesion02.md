# Bitácora de Requerimientos y Auditoría IA — Sesión 02
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Gestión en GitHub Web y Nodos Dinámicos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [No. de Control]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Lista enlazada de tokens en Heap  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue manejar bien los punteros. En `agregar_token` tuve que separar el caso
> de la lista vacía (devolver el nodo nuevo como cabeza) del caso normal, donde hay que
> recorrer la lista hasta el último nodo y enlazar ahí. También tuve que acordarme de poner
> `siguiente = NULL` en el nodo nuevo, porque malloc no inicializa la memoria y quedaría
> basura. Con el campo `lexema[16]` vi que no se puede copiar cualquier cadena: usé
> `strncpy` con 15 caracteres y puse el `'\0'` a mano para no pasarme del arreglo.
> En `liberar_lista` la restricción principal fue guardar el puntero al siguiente nodo
> ANTES de hacer `free`, porque si no se pierde el resto de la lista y además se usaría
> memoria ya liberada (use-after-free).

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
Estoy construyendo una lista enlazada de tokens en el heap. El nodo está definido en
api_sesion02.h así:
  typedef struct NodoToken { char lexema[16]; struct NodoToken *siguiente; } NodoToken;
Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en manejo
seguro de memoria dinámica.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  NodoToken *agregar_token(NodoToken *cabeza, const char *lexema);
  int contar_tokens(NodoToken *cabeza);
  void liberar_lista(NodoToken *cabeza);
Reglas: cada nodo se reserva con malloc; el siguiente del último nodo vale NULL;
agregar_token agrega AL FINAL y devuelve la cabeza (si la lista estaba vacía, devuelve
el nodo nuevo); contar_tokens(NULL) devuelve 0; liberar_lista(NULL) no debe fallar y
no debe usar un nodo después de hacerle free. El lexema se copia sin pasarse de los
16 bytes. Debe manejar malloc devolviendo NULL.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion02.h", sin main(), con
comentarios breves y un análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion02.h"
#include <stdlib.h>
#include <string.h>

NodoToken *agregar_token(NodoToken *cabeza, const char *lexema) {
    NodoToken *nuevo = malloc(sizeof(NodoToken));
    if (nuevo == NULL) {
        return cabeza;
    }
    strncpy(nuevo->lexema, lexema, sizeof(nuevo->lexema) - 1);
    nuevo->lexema[sizeof(nuevo->lexema) - 1] = '\0';
    nuevo->siguiente = NULL;

    if (cabeza == NULL) {
        return nuevo;
    }
    NodoToken *p = cabeza;
    while (p->siguiente != NULL) {
        p = p->siguiente;
    }
    p->siguiente = nuevo;
    return cabeza;
}

int contar_tokens(NodoToken *cabeza) {
    int n = 0;
    for (NodoToken *p = cabeza; p != NULL; p = p->siguiente) {
        n++;
    }
    return n;
}

void liberar_lista(NodoToken *cabeza) {
    while (cabeza != NULL) {
        NodoToken *sig = cabeza->siguiente;
        free(cabeza);
        cabeza = sig;
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Prevención de Use-After-Free y cadenas nulas

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Complejidad:** `agregar_token` es O(n) porque recorre toda la lista hasta el final;
`contar_tokens` es O(n); `liberar_lista` es O(n). El espacio extra es O(1) por llamada
(solo un nodo nuevo en `agregar_token`).

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde que el harness no prueba:
> 1. **`lexema == NULL`:** `strncpy` con NULL provoca un segfault. Lo detecté al revisar
>    qué pasaba con entradas raras, ya que el harness solo prueba casos "felices".
>    Lo solucioné agregando al inicio de `agregar_token`:
>    `if (lexema == NULL) return cabeza;`
> 2. **Lexemas de más de 15 caracteres:** se truncan en silencio y el token cambia sin
>    avisar. Esto en un compilador sería un error difícil de encontrar.
> 3. **`malloc` devuelve NULL:** la IA devuelve la cabeza sin avisar, y quien llama no
>    sabe que el token no se agregó.
> 4. **Eficiencia:** agregar al final recorre toda la lista cada vez. Con muchos tokens
>    conviene guardar un puntero a la cola para que sea O(1).
>
> Conclusión: tener el harness en verde no prueba que el código sea robusto, solo que
> pasa esos 6 casos. Hay que auditar también lo que la IA no consideró.

---
**Firma digital:** [henari2806-ai]
