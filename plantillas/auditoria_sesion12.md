# Bitácora de Requerimientos y Auditoría IA — Sesión 12
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Atributos de Símbolos y Cálculo de Memoria  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Cálculo de anchos de tipo y offsets alineados  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue entender la alineación. Al principio pensaba que bastaba con sumar
> los tamaños, pero si pongo un char (1 byte) y luego un int, el int no puede empezar en
> el byte 1, porque tiene que quedar en un múltiplo de 4. Por eso hay que "redondear hacia
> arriba" el cursor: calculo el resto de dividir el cursor entre el ancho, y si no es 0 le
> sumo lo que falta. Así el char queda en 0, el cursor pasa a 1, y el int se va al offset 4
> (se pierden 3 bytes de relleno). También tuve que distinguir dos cosas que se parecen:
> el offset de la variable (donde empieza) y el valor que devuelve la función (donde
> empieza la SIGUIENTE, o sea offset + ancho). Por último, decidí escribir en el registro
> hasta el final, cuando ya todas las validaciones pasaron, para no dejar el registro a
> medias si algo falla.

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
Estoy calculando el tamaño y el offset de cada variable dentro del marco de pila de un
compilador. El registro está definido en api_sesion12.h:
  typedef struct Simbolo { const char *nombre; const char *tipo; int offset; } Simbolo;
Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en
alineación de memoria y generación de código.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  int ancho_de_tipo(const char *tipo);
  int asignar_offset(Simbolo *simbolo, int cursor);
Reglas: ancho_de_tipo devuelve char=1, short=2, int=4, float=4, double=8, puntero=8 y -1
para cualquier otro tipo; asignar_offset redondea el cursor hacia arriba al siguiente
múltiplo del ancho del tipo, guarda ese offset en el registro y devuelve el cursor donde
empieza la siguiente variable (offset + ancho); el cursor nunca retrocede; un registro
NULL, un cursor negativo o un tipo desconocido devuelven -1 sin escribir en el registro.
Ejemplo: un char en cursor 0 devuelve 1, y un int en cursor 1 queda con offset 4.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion12.h", sin main(), con
comentarios breves y el análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion12.h"
#include <string.h>

int ancho_de_tipo(const char *tipo) {
    static const struct {
        const char *nombre;
        int ancho;
    } tabla[] = {
        {"char", 1}, {"short", 2}, {"int", 4},
        {"float", 4}, {"double", 8}, {"puntero", 8}
    };

    for (size_t i = 0; i < sizeof(tabla) / sizeof(tabla[0]); i++) {
        if (strcmp(tipo, tabla[i].nombre) == 0) {
            return tabla[i].ancho;
        }
    }
    return -1;
}

int asignar_offset(Simbolo *simbolo, int cursor) {
    if (simbolo == NULL || cursor < 0) {
        return -1;
    }
    int ancho = ancho_de_tipo(simbolo->tipo);
    if (ancho < 0) {
        return -1;
    }
    int offset = (cursor + ancho - 1) & ~(ancho - 1);
    simbolo->offset = offset;
    return offset + ancho;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Anchos de int y double, int en cursor 0, char seguido de int, double desde cursor 4 y registro NULL

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Memory leaks:** ninguna de las dos versiones usa malloc, así que no hay nada que
liberar. La tabla de la IA es un arreglo fijo y el ciclo se detiene en su tamaño.

**Nota sobre el crash:** el harness nunca manda un tipo NULL, por eso la versión de la IA
sale en verde. Con un tipo NULL sí se cae (ver el reporte crítico).

**Complejidad:** `ancho_de_tipo` compara contra 6 nombres fijos, así que es O(1) respecto
al número de símbolos (cada comparación recorre el nombre del tipo). `asignar_offset` es
O(1): una división o una operación de bits. El espacio extra es O(1).

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **Tipo NULL:** `ancho_de_tipo(NULL)` llama a `strcmp` con un puntero nulo y el
>    programa se cae. Lo mismo pasa con un registro cuyo campo `tipo` sea NULL, porque
>    `asignar_offset` solo revisa que el registro no sea NULL. Lo detecté revisando qué
>    pasa si `simbolo->tipo` está vacío. Lo solucioné revisando `tipo == NULL` al inicio
>    de `ancho_de_tipo`.
> 2. **Desbordamiento del cursor:** con un cursor cerca de `INT_MAX`, la cuenta
>    `cursor + ancho - 1` se desborda (comportamiento indefinido en C) y el offset puede
>    salir negativo. Eso rompe la regla de que el cursor nunca retrocede. En mi versión
>    rechazo con -1 los cursores demasiado grandes.
> 3. **Tamaño total del marco sin alinear:** la función devuelve el cursor justo después
>    de la última variable. Si la última es un char, el tamaño del marco queda en un
>    número impar, y un compilador real lo redondea al múltiplo de 8 o 16 para mantener
>    alineada la pila. Eso queda fuera del contrato, solo lo documento.
> 4. **Puntero fijo en 8 bytes:** en una máquina de 32 bits un puntero mide 4. El
>    contrato lo fija en 8, así que está bien, pero sería más portable usar
>    `sizeof(void *)`. Lo documento.
> 5. **Mayúsculas y espacios:** "Int" o "int " devuelven -1. Es lo que pide el contrato,
>    pero un compilador real debería decidir cómo tratarlos.
>
> Conclusión: el harness en verde solo prueba tipos válidos y cursores pequeños. La IA
> resolvió bien la alineación con la operación de bits, pero hay que auditar los
> punteros nulos y los límites de los enteros.
---
**Firma digital:** [henari28.06ai]
