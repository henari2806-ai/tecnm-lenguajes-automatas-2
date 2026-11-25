# Bitácora de Requerimientos y Auditoría IA — Sesión 05
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Recorridos del AST y Notación Polaca Inversa  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Recorrido postorden del AST a RPN en un buffer  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue el manejo del buffer. Como no se puede imprimir con printf, tuve que
> ir pegando cada token al final del texto, y el separador era un problema: si ponía un
> espacio después de cada token quedaba un espacio sobrante al final. Lo resolví poniendo
> el espacio ANTES del token, y solo si el buffer ya tenía algo. También tuve que revisar
> que el token cupiera dejando lugar para el '\0' (largo actual + espacio + token < cap),
> porque si no, se escribe fuera del arreglo. Para el orden del recorrido recordé que en
> postorden primero se visitan los dos hijos y hasta el final la raíz, por eso en 3+(5*2)
> sale "3 5 2 * +". Para el árbol nulo, puse buf[0] = '\0' al inicio para que quede
> cadena vacía y no basura.

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
Tengo un árbol sintáctico abstracto (AST) definido en api_sesion05.h con un enum TipoNodo
(NODO_NUM, NODO_BINOP) y los campos tipo, valor, op, izq y der. Quiero convertirlo a
notación polaca inversa (RPN). Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea
que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en
recorridos de árboles y manejo seguro de buffers.

ESPECIFICACIÓN: Implementa exactamente esta firma, sin cambiarla:
  void postorden(ASTNode *n, char *buf, int cap);
Reglas: recorre en postorden (izquierdo, derecho, raíz); escribe los tokens separados por
un solo espacio y sin espacio final; NO uses printf, escribe en el buffer; nunca escribas
más de cap-1 caracteres más el '\0'; un árbol NULL produce cadena vacía; el buffer siempre
debe terminar en '\0'. Ejemplo: 3+(5*2) produce "3 5 2 * +".

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion05.h", sin main(), con
comentarios breves y el análisis de complejidad de la función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion05.h"
#include <stdio.h>
#include <string.h>

static void emitir(ASTNode *n, char *buf, int cap, int *pos) {
    if (n == NULL) {
        return;
    }
    emitir(n->izq, buf, cap, pos);
    emitir(n->der, buf, cap, pos);

    char tok[16];
    if (n->tipo == NODO_NUM) {
        snprintf(tok, sizeof(tok), "%d", n->valor);
    } else {
        tok[0] = n->op;
        tok[1] = '\0';
    }

    int len = (int)strlen(tok);
    int sep = (*pos > 0) ? 1 : 0;
    if (*pos + sep + len >= cap) {
        return;
    }
    if (sep) {
        buf[*pos] = ' ';
        (*pos)++;
    }
    strcpy(buf + *pos, tok);
    *pos += len;
}

void postorden(ASTNode *n, char *buf, int cap) {
    if (buf == NULL || cap <= 0) {
        return;
    }
    buf[0] = '\0';
    int pos = 0;
    emitir(n, buf, cap, &pos);
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Hoja, 3+(5*2), árbol NULL, (1+2)*(3+4) y 8-3

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 5/5 pruebas superadas, todas con `OK`.

**Memory leaks:** `postorden` no usa malloc (el harness arma los nodos en la pila), así
que no hay nada que liberar. Los accesos al buffer están protegidos por la revisión
contra `cap`.

**Complejidad:** el recorrido visita cada nodo una vez, así que es O(n) en nodos. Mi
versión usa `strlen` y `strcat` en cada token, que recorren el buffer completo, por lo
que en el peor caso es O(n·L) con L el largo del texto. La versión de la IA lleva la
posición en `pos` y es O(n) real. El espacio extra es O(h) por la pila de recursión, con h
la altura del árbol.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 5 casos del harness, pero omitió casos borde:
> 1. **Truncado silencioso:** si el texto no cabe en el buffer, la función simplemente
>    deja de agregar el token que no cabe, pero los siguientes tokens más cortos sí
>    pueden caber. Así queda una RPN incompleta y con tokens faltantes en medio, que
>    parece válida. Como la función es `void`, quien la llama no puede saber que se
>    truncó. Lo detecté probando con un buffer de 6 en 3+(5*2). Una mejora sería devolver
>    un código de error o dejar de escribir en cuanto algo no cabe.
> 2. **Nodo operación con hijo NULL:** si un `NODO_BINOP` tiene un hijo NULL, el
>    recorrido lo salta sin avisar y produce un RPN inválido, por ejemplo "3 +" sin su
>    segundo operando.
> 3. **Operador inválido:** si `op` vale '\0' en un nodo operación, el token queda como
>    cadena vacía y se escribe un espacio de más.
> 4. **Árboles muy profundos:** la recursión puede desbordar la pila si el árbol es una
>    lista larga (por ejemplo miles de sumas encadenadas).
>
> Conclusión: el harness en verde solo comprueba 5 casos "felices". El buffer lo cuidó
> bien la IA, pero hay que auditar qué pasa cuando el texto no cabe y cuando el árbol
> está mal formado.
