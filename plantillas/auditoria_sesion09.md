# Bitácora de Requerimientos y Auditoría IA — Sesión 09
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Reporte Formal de Errores Semánticos y Diagnóstico  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Formateador de diagnósticos con línea y columna  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue respetar el tamaño del buffer. Al principio pensé en armar el texto
> con `sprintf` o `strcat`, pero esas funciones no revisan el límite y pueden escribir
> fuera del arreglo. Por eso usé `snprintf`, que corta el texto en `cap-1` caracteres y
> siempre pone el `'\0'`. Con `cap = 10` el mensaje se corta en "main.c:14", justo lo que
> pide el caso 4. También tuve que cuidar el formato exacto: dos puntos entre archivo,
> línea y columna, y luego `": error: "` con un espacio después de cada dos puntos y
> sin punto al final. Revisé al inicio que `salida` no sea NULL y que `cap` sea mayor que 0,
> porque con un `cap` negativo la conversión a `size_t` lo vuelve un número enorme.

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
Estoy haciendo el módulo de diagnósticos de un compilador. La interfaz está en
api_sesion09.h. Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier
advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en
diagnósticos de errores y manejo seguro de buffers.

ESPECIFICACIÓN: Implementa exactamente esta firma, sin cambiarla:
  void reportar(const char *archivo, int linea, int columna, const char *mensaje,
                char *salida, int cap);
Reglas: escribe en salida el formato exacto "archivo:linea:columna: error: mensaje", como
GCC, sin espacios extra ni punto final; nunca escribas más de cap-1 caracteres más el
'\0'; el resultado siempre termina en '\0'; si no cabe, se trunca. Ejemplo:
reportar("main.c", 14, 5, "tipo incompatible", ...) produce
"main.c:14:5: error: tipo incompatible".

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion09.h", sin main(), con
comentarios breves y el análisis de complejidad de la función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion09.h"
#include <stdio.h>

void reportar(const char *archivo, int linea, int columna, const char *mensaje, char *salida, int cap) {
    if (salida == NULL) {
        return;
    }
    snprintf(salida, (size_t)cap, "%s:%d:%d: error: %s", archivo, linea, columna, mensaje);
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Formato normal, línea y columna en cero, y truncado con cap = 10

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 4/4 pruebas superadas, todas con `OK`.

**Nota sobre accesos fuera de límites:** el harness solo prueba buffers con `cap` válido,
por eso la versión de la IA sale en verde. Con un `cap` negativo la IA sí puede escribir
fuera del buffer (ver el reporte crítico). Mi versión lo evita. No uso memoria dinámica,
así que no hay fugas.

**Complejidad:** `reportar` es O(n) en el largo de los textos que copia (archivo y
mensaje), con un máximo de `cap` caracteres escritos. El espacio extra es O(1).

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 4 casos del harness, pero omitió casos borde:
> 1. **`cap` negativo o cero:** la IA convierte `cap` a `size_t` sin revisarlo. Un `cap`
>    negativo se vuelve un número enorme y `snprintf` cree que tiene todo ese espacio,
>    así que escribe más allá del buffer. Lo detecté revisando la conversión a `size_t`
>    y probando `reportar(..., buf, -1)`. Lo solucioné con `if (salida == NULL || cap <= 0) return;`.
> 2. **`archivo` o `mensaje` en NULL:** imprimir un puntero nulo con `%s` es
>    comportamiento indefinido y puede tronar el compilador justo cuando intenta reportar
>    un error. En mi versión los reemplazo por "?".
> 3. **Truncado silencioso:** como la función es `void`, quien la llama no sabe si el
>    mensaje se cortó. Un diagnóstico cortado puede cambiar su significado. Una mejora
>    sería devolver la cantidad de caracteres que se necesitaban, como hace `snprintf`.
> 4. **Solo nivel `error`:** el formato tiene el nivel fijo. Un compilador real también
>    necesita `warning` y `note`. El contrato no lo pide, así que solo lo documento.
>
> Conclusión: el harness en verde solo prueba entradas válidas. En un compilador la
> función de errores es la que se ejecuta cuando algo ya salió mal, por eso no debe
> fallar ella misma ante datos inesperados.
