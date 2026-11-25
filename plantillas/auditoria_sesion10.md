# Bitácora de Requerimientos y Auditoría IA — Sesión 10
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Hito 1: Mini-Evaluador de Expresiones y AST Completo  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Mini-evaluador de expresiones con precedencia y paréntesis  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue entender cómo se consigue la precedencia sin escribir reglas
> especiales. La idea es que `expr` maneja la suma y la resta, pero cada uno de sus
> operandos lo pide a `termino`, que maneja la multiplicación y la división, y esos
> operandos los pide a `primaria`, que lee números o paréntesis. Así, en 2+3*4 el 3*4
> se resuelve primero porque queda más abajo en las llamadas. Otro problema fue la
> recursión entre `expr` y `primaria` para los paréntesis: tuve que poner el prototipo
> de `expr` antes, porque `primaria` la llama antes de que esté definida. También me
> costó acordarme de revisar `p->error` después de cada llamada, para no seguir
> calculando con un valor que ya salió mal, y de comprobar el divisor antes de dividir
> para que 10/0 dé error 1 y no tire el programa. Por último, 2+ da error 2 porque
> `primaria` no encuentra ni un número ni un paréntesis después del +.

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
Estoy haciendo el Hito 1: un mini-evaluador que lee una expresión aritmética desde texto.
La interfaz está en api_sesion10.h. Se compila con gcc -std=c11 -Wall -Wextra -Werror, o
sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en
analizadores sintácticos descendentes recursivos y manejo de errores.

ESPECIFICACIÓN: Implementa exactamente esta firma, sin cambiarla:
  long evaluar_expresion(const char *fuente, int *error);
Reglas: lee enteros, los operadores + - * / y paréntesis, ignorando espacios; sin variables
ni menos unario; respeta la precedencia (* y / antes que + y -) y asocia a la izquierda,
así que 10-2-3 da 5; *error vale 0 si todo salió bien, 1 si hay división entre cero y 2 si
la sintaxis es inválida (por ejemplo "2+", "(2+3" o texto sobrante); comprueba el divisor
antes de dividir; no debe tener main().

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion10.h", con comentarios breves
y el análisis de complejidad de la función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion10.h"
#include <ctype.h>

typedef struct { const char *s; int error; } Lector;

static void espacios(Lector *l) {
    while (isspace((unsigned char)*l->s)) {
        l->s++;
    }
}

static long expresion(Lector *l);

static long primaria(Lector *l) {
    espacios(l);
    if (*l->s == '(') {
        l->s++;
        long v = expresion(l);
        espacios(l);
        if (l->error) {
            return 0;
        }
        if (*l->s != ')') {
            l->error = 2;
            return 0;
        }
        l->s++;
        return v;
    }
    if (!isdigit((unsigned char)*l->s)) {
        l->error = 2;
        return 0;
    }
    long v = 0;
    while (isdigit((unsigned char)*l->s)) {
        v = v * 10 + (*l->s - '0');
        l->s++;
    }
    return v;
}

static long termino(Lector *l) {
    long v = primaria(l);
    for (;;) {
        espacios(l);
        char op = *l->s;
        if (l->error || (op != '*' && op != '/')) {
            return v;
        }
        l->s++;
        long d = primaria(l);
        if (l->error) {
            return 0;
        }
        if (op == '*') {
            v *= d;
        } else if (d == 0) {
            l->error = 1;
            return 0;
        } else {
            v /= d;
        }
    }
}

static long expresion(Lector *l) {
    long v = termino(l);
    for (;;) {
        espacios(l);
        char op = *l->s;
        if (l->error || (op != '+' && op != '-')) {
            return v;
        }
        l->s++;
        long d = termino(l);
        if (l->error) {
            return 0;
        }
        v = (op == '+') ? v + d : v - d;
    }
}

long evaluar_expresion(const char *fuente, int *error) {
    if (fuente == NULL) {
        *error = 2;
        return 0;
    }
    Lector l = { fuente, 0 };
    long v = expresion(&l);
    espacios(&l);
    if (l.error == 0 && *l.s != '\0') {
        l.error = 2;
    }
    *error = l.error;
    return l.error ? 0 : v;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* 2+3*4, (2+3)*4, 10/0, 2+, "  7 " y 5-10

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Memory leaks:** el evaluador no usa malloc, todo vive en la pila, así que no hay
nada que liberar. El analizador solo avanza el puntero por el texto y se detiene en el
`'\0'`, por eso no se sale de la cadena.

**Complejidad:** el analizador lee cada carácter una sola vez, así que es O(n) en el
largo del texto. El espacio extra es O(p), con p la profundidad máxima de paréntesis,
por la pila de la recursión.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **`error == NULL`:** la IA escribe `*error = ...` sin revisar si el puntero es
>    nulo. Con `evaluar_expresion("2+3", NULL)` el programa se cae. Lo detecté revisando
>    el código de `evaluar_expresion` y comparándolo con el contrato de sesiones
>    anteriores. En mi versión uso una variable interna cuando `error` es NULL.
> 2. **Números demasiado grandes:** al leer los dígitos hace `v * 10 + dígito` sin
>    revisar el límite. Con un número de más de 19 dígitos, `long` se desborda, lo cual
>    es comportamiento indefinido en C, y la función devuelve basura sin avisar. Lo
>    mismo pasa con `v * d` y `v + d` si el resultado no cabe. Lo documento, pero no
>    lo corregí.
> 3. **Paréntesis muy anidados:** con miles de `(` seguidos la recursión llena la pila
>    y el programa se cae. Un compilador real debería poner un límite de profundidad y
>    reportar error 2.
> 4. **Espacios dentro de un número:** `"1 2"` da error 2 (texto sobrante), lo cual es
>    correcto, pero la IA acepta también tabuladores y saltos de línea como espacios.
>    Es un detalle que conviene decidir en la especificación.
>
> Conclusión: el harness en verde solo prueba 6 expresiones sencillas. El analizador de
> la IA es correcto en precedencia y paréntesis, pero hay que auditar los límites
> numéricos, la profundidad de la recursión y los punteros nulos.
**Firma digital:** [Usuario de GitHub]
