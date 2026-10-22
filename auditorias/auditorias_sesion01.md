# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:**Ricardo Hernandez Naranjo  
**Matrícula:** 24531374  
**Track:** [ C | Rust ]  

---

# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Lenguajes y Autómatas II (SCD-1016)
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)
**Estudiante:** Ricardo Hernandez Naranjo
**Matrícula:** 24531374
**Track:** [ C ]

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Inversión de buffer in-place (char*)
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Al codificarlo a mano identifiqué varios problemas:
> - **El puntero no conoce su longitud.** Un `char *` solo guarda una dirección, así que hay que recibir `len` o calcularlo con `strlen`, que lee fuera de límites si el buffer no termina en `'\0'`.
> - **Error off-by-one.** Mi primer intento apuntaba el puntero derecho a `buf + len` (el `'\0'`) en lugar de `buf + len - 1`, y el terminador terminaba en la primera posición.
> - **Longitud impar.** El carácter central no se intercambia. Con `left < right` el ciclo se detiene solo, pero con `left != right` el ciclo es infinito en longitudes pares.
> - **Literales de cadena.** Pasar `"hola"` directamente provoca *segmentation fault* porque el literal está en memoria de solo lectura. Hay que usar un arreglo (`char s[] = "hola";`).
> - **NULL y buffer vacío.** Hay que validarlos antes de desreferenciar el puntero.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Punteros, O(n) tiempo, O(1) memoria |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
Contexto: Estoy trabajando en C99, compilado con GCC (-std=c99 -Wall -Wextra),
sin bibliotecas fuera de la estándar.

Rol: Actúa como ingeniero de software de sistemas y compiladores.

Especificación: Implementa la función
    void reverse_buffer(char *buf, size_t len);
que invierta el buffer in-place usando aritmética de punteros (dos punteros,
uno al inicio y otro al final). Requisitos:
  - Complejidad O(n) en tiempo y O(1) en memoria (sin buffers auxiliares).
  - Debe manejar buf == NULL, len == 0 y len == 1 sin fallar.
  - Debe funcionar con longitudes pares e impares.
  - No debe leer ni escribir fuera de los límites [buf, buf + len).
  - No usar strlen ni funciones de string.h dentro de la función.

Output: Entrega únicamente código modular (la función y una función main de
prueba separada) y un breve análisis de complejidad temporal y espacial.
Sin explicaciones adicionales.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include <stddef.h>
#include <stdio.h>
#include <string.h>

void reverse_buffer(char *buf, size_t len) {
    if (buf == NULL || len < 2) {
        return;
    }

    char *left = buf;
    char *right = buf + len - 1;

    while (left < right) {
        char tmp = *left;
        *left = *right;
        *right = tmp;
        left++;
        right--;
    }
}

int main(void) {
    char a[] = "abcde";   /* longitud impar */
    char b[] = "abcd";    /* longitud par   */
    char c[] = "";        /* vacío          */

    reverse_buffer(a, strlen(a));
    reverse_buffer(b, strlen(b));
    reverse_buffer(c, strlen(c));
    reverse_buffer(NULL, 0);

    printf("%s\n%s\n[%s]\n", a, b, c);
    return 0;
}

/*
 * Complejidad temporal: O(n) -> se hacen n/2 intercambios.
 * Complejidad espacial: O(1) -> solo se usan dos punteros y una variable tmp.
 */
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Valores NULL, buffers vacios y longitudes impares
gcc -std=c99 -Wall -Wextra -g -fsanitize=address,undefined -o reverse_san reverse.c && ./reverse_san
```
[PASS] longitud par                 -> "dcba" (esperado "dcba")
[PASS] longitud impar               -> "edcba" (esperado "edcba")
[PASS] buffer vacio                 -> "" (esperado "")
[PASS] un solo caracter             -> "a" (esperado "a")
[PASS] dos caracteres               -> "ba" (esperado "ba")
[PASS] palindromo                   -> "racecar" (esperado "racecar")
[PASS] con espacio                  -> "odnuM aloH" (esperado "odnuM aloH")
[PASS] arreglo (no literal)         -> "aloh" (esperado "aloh")
[PASS] puntero NULL, len 0          -> sin crash
[PASS] puntero NULL, len 5          -> sin crash

Resultado: 10/10 pruebas superadas
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (AddressSanitizer/UBSan sin reportes).
* [x] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA resolvió bien la lógica de los dos punteros, pero omitió que la función **confía ciegamente en el parámetro `len`**: si quien la llama pasa un `len` mayor al tamaño real del buffer, escribe fuera de límites, y el modelo no lo advirtió ni lo documentó. Tampoco mencionó el caso de los **literales de cadena**: `reverse_buffer("hola", 4)` compila, pero falla en ejecución porque el literal está en memoria de solo lectura.
>
> Cómo lo solucioné:
> 1. Agregué un comentario de precondición: `buf` debe apuntar a memoria modificable de al menos `len` bytes.
> 2. Añadí al test harness un caso con arreglo (`char s[] = "hola";`) en lugar de literal.
> 3. Verifiqué con `gcc -fsanitize=address,undefined` y con `valgrind` que no hubiera accesos inválidos.
> 4. Confirmé que la guarda `len < 2` cubre `len == 0`, ya que sin ella `buf + len - 1` con `size_t` causaría un desbordamiento por debajo.

---

**Firma digital:** [henari2806-ai](https://github.com/henari2806-ai)
