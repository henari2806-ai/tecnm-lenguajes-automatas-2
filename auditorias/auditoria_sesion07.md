# Bitácora de Requerimientos y Auditoría IA — Sesión 07
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Evaluación de Expresiones Aritméticas y Casos Límite  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Evaluador recursivo con detección de errores  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue manejar el error en la recursión. Si el hijo izquierdo da error y
> sigo evaluando el derecho, el segundo puede sobrescribir el código del primero. Por eso
> revisé `*error` después de cada hijo y salí de inmediato si valía distinto de 0. Otro
> detalle fue la división: en C dividir entre cero hace que el programa se caiga (error de
> punto flotante, SIGFPE), así que tuve que comprobar `b == 0` ANTES de hacer `a / b` y
> también antes de `a % b`. También me di cuenta de que `evaluar` no debe reiniciar el
> error en cada llamada recursiva, porque borraría un error ya encontrado. Para eso separé
> una función auxiliar `calcular`, y `evaluar` solo pone el error en 0 al inicio.

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
Tengo un árbol sintáctico abstracto (AST) definido en api_sesion07.h con un enum TipoNodo
(NODO_NUM, NODO_BINOP) y los campos tipo, valor, op, izq y der. Quiero evaluarlo de forma
recursiva. Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier
advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en
evaluación de expresiones y manejo de errores sin fallos de ejecución.

ESPECIFICACIÓN: Implementa exactamente esta firma, sin cambiarla:
  int evaluar(ASTNode *n, int *error);
Reglas: soporta los operadores + - * / %; *error vale 0 si todo salió bien, 1 si hay
división o módulo entre cero y 2 si el nodo es inválido (árbol NULL u operador
desconocido); comprueba el divisor ANTES de dividir; evaluar(NULL, &e) no debe caerse; un
error en un hijo se propaga hacia arriba y el padre no se calcula.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion07.h", sin main(), con
comentarios breves y el análisis de complejidad de la función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion07.h"

int evaluar(ASTNode *n, int *error) {
    if (n == NULL) {
        *error = 2;
        return 0;
    }
    if (n->tipo == NODO_NUM) {
        return n->valor;
    }

    int a = evaluar(n->izq, error);
    int b = evaluar(n->der, error);
    if (*error != 0) {
        return 0;
    }

    switch (n->op) {
        case '+': return a + b;
        case '-': return a - b;
        case '*': return a * b;
        case '/':
            if (b == 0) {
                *error = 1;
                return 0;
            }
            return a / b;
        case '%':
            if (b == 0) {
                *error = 1;
                return 0;
            }
            return a % b;
        default:
            *error = 2;
            return 0;
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* (2+3)*4, 7/0, 7%0, árbol NULL, 10-2-3 y (8/2)+1

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Complejidad:** `evaluar` visita cada nodo una vez, así que es O(n) en el número de
nodos. El espacio extra es O(h) por la pila de la recursión, con h la altura del árbol.
No usa memoria dinámica, por eso no hay fugas.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **Un error del hijo derecho sobrescribe al del izquierdo:** la IA evalúa los dos
>    hijos y revisa el error hasta el final. Si el izquierdo da error 1 (7/0) y el derecho
>    es un árbol NULL (error 2), el resultado final es 2 y se pierde el error real.
>    Además evalúa el derecho aunque el izquierdo ya falló, y el contrato dice que un
>    error en un hijo impide seguir. Lo detecté armando a mano el árbol (7/0) + NULL.
>    Lo solucioné revisando `*error` después de cada hijo, como en mi versión.
> 2. **No reinicia `*error`:** si quien llama reutiliza la variable con un error de una
>    evaluación anterior, la IA devuelve 0 aunque el árbol esté bien, porque nunca pone
>    `*error = 0` al inicio. Lo solucioné con la función `evaluar` que lo reinicia y una
>    auxiliar `calcular` para la recursión.
> 3. **`error == NULL`:** si se llama `evaluar(n, NULL)`, la IA escribe en un puntero
>    nulo y el programa se cae. En mi versión uso una variable local en ese caso.
> 4. **Desbordamiento de enteros (no lo corregí, solo lo documento):** `INT_MIN / -1`
>    provoca el mismo fallo que dividir entre cero, y `a + b` o `a * b` con números
>    grandes es comportamiento indefinido en C. Ni la IA ni mi versión lo validan.
>
> Conclusión: el harness en verde solo comprueba 6 casos con un solo error a la vez. Un
> compilador real necesita cuidar qué error se reporta y qué pasa con los límites de los
> enteros.

---
**Firma digital:** [Usuario de GitHub]
