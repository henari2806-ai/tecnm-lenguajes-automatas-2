# Bitácora de Requerimientos y Auditoría IA — Sesión 03
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Google Colab, Mágicos y Flujo de Entrega  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Escribir y leer archivos desde C con códigos de retorno  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue contar bien las líneas. Al principio solo contaba los `'\n'`, y con
> el archivo "a\nb" daba 1 en lugar de 2, porque la última línea no termina en salto.
> Lo resolví guardando el último carácter leído: si no es `'\n'`, sumo una línea más.
> Inicié esa variable en `'\n'` para que un archivo vacío dé 0 y no 1. También tuve que
> acordarme de que `fopen` devuelve NULL cuando falla (ruta inexistente), y de reportarlo
> con -1 en vez de dejar que el programa se caiga. Por último, usé `int` y no `char` para
> guardar lo que devuelve `fgetc`, porque `EOF` vale -1 y no cabe bien en un `char`.

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
Voy a escribir y leer archivos de texto con la biblioteca estándar (stdio.h). Las firmas
están en api_sesion03.h. Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que
cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en manejo
seguro de archivos y códigos de retorno.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  int escribir_texto(const char *ruta, const char *texto);
  int contar_lineas(const char *ruta);
Reglas: usa fopen/fgetc y cierra siempre el archivo; escribir_texto devuelve 0 si todo
salió bien y -1 si la ruta no se puede abrir; contar_lineas devuelve el número de líneas
y -1 si el archivo no existe, sin abortar. Una última línea sin salto de línea también
cuenta (por ejemplo "a\nb" son 2 líneas) y un archivo vacío tiene 0 líneas.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion03.h", sin main(), con
comentarios breves y el análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion03.h"
#include <stdio.h>

int escribir_texto(const char *ruta, const char *texto) {
    FILE *f = fopen(ruta, "w");
    if (f == NULL) {
        return -1;
    }
    fputs(texto, f);
    fclose(f);
    return 0;
}

int contar_lineas(const char *ruta) {
    FILE *f = fopen(ruta, "r");
    if (f == NULL) {
        return -1;
    }
    int lineas = 0;
    int c;
    int previo = 0;
    while ((c = fgetc(f)) != EOF) {
        if (c == '\n') {
            lineas++;
        }
        previo = c;
    }
    if (previo != 0 && previo != '\n') {
        lineas++;
    }
    fclose(f);
    return lineas;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Archivo sin salto final, archivo vacío y ruta inexistente

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Complejidad:** `escribir_texto` es O(n) en el largo del texto; `contar_lineas` es O(n)
en el número de caracteres del archivo, con espacio extra O(1) porque lee carácter por
carácter sin guardar el archivo en memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió varios casos borde:
> 1. **`ruta` o `texto` igual a NULL:** `fopen(NULL, ...)` o `fputs(NULL, ...)` pueden
>    provocar un segfault. Lo detecté probando entradas raras fuera del harness.
>    Lo solucioné validando al inicio: `if (ruta == NULL || texto == NULL) return -1;`
> 2. **No revisa el resultado de `fputs` ni de `fclose`:** si el disco está lleno o falla
>    la escritura, la función devuelve 0 (éxito) aunque el archivo quedó incompleto. Para
>    un pipeline de compilación esto es grave, porque el siguiente paso trabajaría con un
>    archivo corrupto. Lo solucioné comprobando que `fputs` no devuelva `EOF` y que
>    `fclose` devuelva 0.
> 3. **Archivo que contiene un carácter con valor 0:** la IA usó 0 como "nada leído" en
>    `previo`, y eso confunde ese caso con un archivo vacío. Lo cambié por inicializar
>    `previo` en `'\n'`, como en mi versión.
>
> Conclusión: el harness en verde solo prueba 6 casos. Un código que reporta éxito cuando
> en realidad falló rompe la idea de la sesión, que es que una máquina de compilación
> sepa de verdad si un paso terminó bien o mal.
---
**Firma digital:** [henari2806-ai]
