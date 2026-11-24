# Bitácora de Requerimientos y Auditoría IA — Sesión 04
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Anatomía de un Árbol Sintáctico Abstracto (AST)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Nodo binario de AST en Heap  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue entender la recursión en `evaluar`. Tuve que separar el caso base
> (un nodo número, que solo devuelve su valor) del caso recursivo (un nodo operación,
> que primero evalúa el hijo izquierdo y el derecho y luego aplica el operador). Al
> principio olvidaba poner `izq` y `der` en NULL en los nodos número, y como malloc no
> inicializa la memoria quedaban punteros basura. También tuve que cuidar el orden en
> `crear_binario`: en una resta, si invierto izquierdo y derecho, 10-4 daría -6 en lugar
> de 6. Por último, revisé `n == NULL` al inicio de `evaluar` para no tener un fallo de
> segmentación con la entrada vacía.

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
Estoy modelando expresiones aritméticas como un árbol sintáctico abstracto (AST). El nodo
está definido en api_sesion04.h con un enum TipoNodo (NODO_NUM, NODO_BINOP) y los campos
tipo, valor, op, izq y der. Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que
cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en
estructuras recursivas y manejo seguro de memoria dinámica.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  ASTNode *crear_numero(int v);
  ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der);
  int evaluar(ASTNode *n);
Reglas: cada nodo se reserva con malloc; usa el campo tipo para distinguir número de
operación; crear_binario respeta el orden izquierdo/derecho; evaluar evalúa ambos hijos
antes de aplicar el operador (+, - y * únicamente); evaluar(NULL) devuelve -1 sin
provocar fallo de segmentación. Debe manejar malloc devolviendo NULL.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion04.h", sin main(), con
comentarios breves y el análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion04.h"
#include <stdlib.h>

ASTNode *crear_numero(int v) {
    ASTNode *n = malloc(sizeof(ASTNode));
    if (n == NULL) {
        return NULL;
    }
    n->tipo = NODO_NUM;
    n->valor = v;
    n->op = 0;
    n->izq = NULL;
    n->der = NULL;
    return n;
}

ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der) {
    ASTNode *n = malloc(sizeof(ASTNode));
    if (n == NULL) {
        return NULL;
    }
    n->tipo = NODO_BINOP;
    n->valor = 0;
    n->op = op;
    n->izq = izq;
    n->der = der;
    return n;
}

int evaluar(ASTNode *n) {
    if (n == NULL) {
        return -1;
    }
    if (n->tipo == NODO_NUM) {
        return n->valor;
    }
    int a = evaluar(n->izq);
    int b = evaluar(n->der);
    switch (n->op) {
        case '+': return a + b;
        case '-': return a - b;
        case '*': return a * b;
        default:  return 0;
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Evaluación de 3+5*2 y 10-4, orden de hijos, operador guardado y evaluar(NULL)

* [x] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Nota sobre la casilla de memory leaks:** no la marqué porque el contrato de esta sesión
no incluye una función para liberar el árbol. El harness crea nodos con malloc y nunca
los libera, así que una herramienta como AddressSanitizer reportaría fugas.

**Complejidad:** `crear_numero` y `crear_binario` son O(1). `evaluar` es O(n) en el
número de nodos, porque visita cada nodo una vez. El espacio extra es O(h), con h la
altura del árbol, por la pila de la recursión.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió varios casos borde:
> 1. **Operador desconocido:** con `default: return 0` un operador inválido (por ejemplo
>    '/' o '?') devuelve 0 en silencio, y ese valor falso se propaga por todo el árbol.
>    Lo detecté probando a mano `crear_binario('/', ...)`. Lo cambié para devolver -1,
>    como en mi versión.
> 2. **-1 es ambiguo:** `evaluar(NULL)` devuelve -1, pero -1 también es un resultado
>    válido (por ejemplo 3-4). Quien llama no puede distinguir un error de un valor real.
>    Es una limitación del contrato; una mejora sería devolver el error por un puntero.
> 3. **Hijo NULL dentro de un binario:** `crear_binario('+', NULL, ...)` se acepta y
>    después `evaluar` calcula con -1 como si fuera un número, dando un resultado
>    incorrecto sin avisar.
> 4. **Fuga si falla malloc:** si `crear_binario` no logra reservar memoria, devuelve NULL
>    y los hijos que recibió quedan sin liberar.
> 5. **No hay función para liberar el árbol:** el contrato no la pide y la IA no la
>    propuso, pero un compilador real necesita una `liberar_ast` recursiva que libere los
>    hijos antes que el padre.
>
> Conclusión: el harness en verde solo comprueba 6 casos "felices". La IA resolvió bien
> la recursión, pero hay que auditar qué pasa con entradas inválidas y con la memoria.

---
**Firma digital:** [henari2806-ai]
