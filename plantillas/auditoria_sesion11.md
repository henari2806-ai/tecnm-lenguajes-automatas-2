# Bitácora de Requerimientos y Auditoría IA — Sesión 11
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Arquitectura de la Tabla de Símbolos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [No. de Control]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tabla hash con encadenamiento para la tabla de símbolos  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue entender qué pasa cuando dos nombres caen en la misma cubeta. Al
> principio pensé que había que sobrescribir, pero la idea es encadenar: el nodo nuevo
> apunta (`sig`) a lo que ya estaba en la cubeta y luego la cubeta apunta al nodo nuevo,
> así no se pierde nada. El orden de esas dos líneas importa: si primero hago
> `tabla[i] = nuevo`, pierdo la referencia a la cadena vieja. También tuve que revisar
> si el nombre ya existía ANTES de hacer malloc, para no reservar memoria que luego se
> queda sin usar. Para `hash_djb2` usé `unsigned int`, porque la multiplicación por 33 se
> desborda y con un tipo con signo eso es comportamiento indefinido; con `unsigned` da la
> vuelta de forma definida. Además el campo `nombre` solo mide 16, así que usé `strlen`
> para rechazar nombres de más de 15 letras en vez de cortarlos, porque un nombre cortado
> ya sería otro nombre distinto.

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
Estoy construyendo la tabla de símbolos de un compilador como tabla hash con
encadenamiento. El nodo está definido en api_sesion11.h:
  typedef struct Simbolo { char nombre[16]; int tipo; int offset; struct Simbolo *sig; } Simbolo;
Se compila con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en tablas
hash y manejo seguro de memoria dinámica.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  unsigned int hash_djb2(const char *clave);
  int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas, const char *nombre,
                       int tipo, int offset);
Reglas: hash_djb2 empieza en 5381 y en cada byte hace h = h * 33 + byte (hash_djb2("")
vale 5381 y hash_djb2("abc") vale 193485963); insertar_simbolo calcula la cubeta con
hash % cubetas; cada símbolo nuevo se reserva con malloc; las colisiones se encadenan y
nunca se sobrescribe una entrada; si el nombre ya existe devuelve 0 sin crear nodo; devuelve
1 si insertó; devuelve -1 con tabla NULL, nombre NULL o cubetas igual a 0. Debe manejar
malloc devolviendo NULL.

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion11.h", sin main(), con
comentarios breves y el análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion11.h"
#include <stdlib.h>
#include <string.h>

unsigned int hash_djb2(const char *clave) {
    unsigned int h = 5381;
    int c;
    while ((c = *clave++) != 0) {
        h = h * 33 + c;
    }
    return h;
}

int insertar_simbolo(Simbolo *tabla[], unsigned int cubetas, const char *nombre, int tipo, int offset) {
    if (tabla == NULL || nombre == NULL || cubetas == 0) {
        return -1;
    }

    unsigned int i = hash_djb2(nombre) % cubetas;

    for (Simbolo *p = tabla[i]; p != NULL; p = p->sig) {
        if (strcmp(p->nombre, nombre) == 0) {
            return 0;
        }
    }

    Simbolo *nuevo = malloc(sizeof(Simbolo));
    if (nuevo == NULL) {
        return -1;
    }
    strncpy(nuevo->nombre, nombre, sizeof(nuevo->nombre) - 1);
    nuevo->nombre[sizeof(nuevo->nombre) - 1] = '\0';
    nuevo->tipo = tipo;
    nuevo->offset = offset;
    nuevo->sig = tabla[i];
    tabla[i] = nuevo;
    return 1;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* hash de "" y "abc", nombre repetido, colisión en una cubeta, offset en la cadena y tabla NULL

* [x] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Salida del esqueleto antes de implementar:** (pegar aquí la salida con `FALLA` que
dio la primera ejecución, antes de escribir el código).

**Nota sobre la casilla de memory leaks:** no la marqué porque el contrato no incluye
una función para liberar la tabla. El harness crea nodos con malloc y nunca los libera,
así que una herramienta como AddressSanitizer reportaría fugas. Los accesos al arreglo
de cubetas sí están protegidos porque el índice sale de `hash % cubetas`, que siempre es
menor que `cubetas`.

**Complejidad:** `hash_djb2` es O(L), con L el largo del nombre. `insertar_simbolo` es
O(L + k), con k el largo de la cadena de esa cubeta. En promedio, con buena dispersión y
pocas colisiones, k es chico y queda casi O(L). En el peor caso, con todos los nombres en
una cubeta, k = n y queda O(n). El espacio extra es O(1) por inserción (un nodo).

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **Nombres de más de 15 caracteres:** la IA calcula el hash con el nombre completo
>    pero guarda solo los primeros 15 caracteres. Al buscar duplicados compara el nombre
>    completo contra el guardado ya cortado, y nunca coinciden. Resultado: el mismo nombre
>    largo se inserta una y otra vez sin que se detecte el duplicado, y dos nombres largos
>    distintos que empiezan igual quedan guardados como el mismo texto. Lo detecté
>    pensando en el `strncpy` y comprobando que el nombre largo repetido devuelve 1 en las
>    dos inserciones. Lo solucioné rechazando con -1 los nombres que no caben, como en mi
>    versión.
> 2. **`hash_djb2(NULL)`:** la IA no revisa el puntero y la función se cae. En mi versión
>    devuelve 5381.
> 3. **Letras con acento o ñ:** la IA lee cada byte como `int` desde un `char` con signo,
>    así que los bytes mayores a 127 salen negativos y el hash cambia según el compilador.
>    Yo lo corregí con `(unsigned char)`.
> 4. **No hay función para liberar la tabla:** el contrato no la pide y la IA no la
>    propuso, pero un compilador real necesita recorrer todas las cadenas y hacer `free`
>    de cada nodo.
> 5. **Tabla sin inicializar:** si quien llama no puso las cubetas en NULL, el recorrido
>    de la cadena lee punteros basura. El contrato lo supone, pero no se puede verificar
>    desde dentro de la función.
>
> Conclusión: el harness en verde solo prueba nombres cortos. La IA resolvió bien las
> colisiones, pero el límite de 16 caracteres del struct afecta la detección de
> duplicados y hay que auditarlo.
