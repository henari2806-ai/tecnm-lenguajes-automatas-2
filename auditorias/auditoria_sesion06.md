# Bitácora de Requerimientos y Auditoría IA — Sesión 06
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Sistema de Tipos Primitivos y Verificación Semántica  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Ricardo Hernández Naranjo  
**Matrícula:** [24531374]  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Tabla de compatibilidad de tipos  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Lo más difícil fue ver que TIPO_ERROR vale 0, así que es el número más chico del enum.
> Si solo comparaba `destino >= origen`, el caso compatible(INT, ERROR) daba 1 (porque
> 2 >= 0) y eso es incorrecto. Por eso tuve que revisar el error ANTES de comparar. Otro
> detalle fue la dirección de la regla: se puede guardar un valor angosto en un destino
> ancho (char en int, int en float), pero no al revés, porque se perdería información
> (por ejemplo, la parte decimal al pasar de float a int). En `nombre_tipo` usé un
> `switch` con un `default` para que un valor fuera del enum no devuelva basura.

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
Estoy haciendo la verificación semántica de un compilador. El enum Tipo está definido en
api_sesion06.h: TIPO_ERROR = 0, TIPO_CHAR = 1, TIPO_INT = 2, TIPO_FLOAT = 3. Se compila
con gcc -std=c11 -Wall -Wextra -Werror, o sea que cualquier advertencia es error.

ROL: Actúa como ingeniero de software de sistemas y compiladores, con énfasis en sistemas
de tipos y verificación semántica.

ESPECIFICACIÓN: Implementa exactamente estas firmas, sin cambiarlas:
  int compatible(Tipo destino, Tipo origen);
  const char *nombre_tipo(Tipo t);
Reglas: la jerarquía es CHAR < INT < FLOAT; compatible devuelve 1 si el destino es de igual
o mayor rango que el origen y 0 si no (asignar a un tipo más angosto requiere cast
explícito); TIPO_ERROR nunca es compatible con nada, ni como destino ni como origen;
nombre_tipo devuelve "char", "int", "float" o "error".

OUTPUT: Entrégame solo el archivo .c con #include "api_sesion06.h", sin main(), con
comentarios breves y el análisis de complejidad de cada función.
```

---

## 3. Código Devuelto por el Modelo
```c
// Código devuelto por el LLM
#include "api_sesion06.h"

int compatible(Tipo destino, Tipo origen) {
    if (destino == TIPO_ERROR || origen == TIPO_ERROR) {
        return 0;
    }
    return destino >= origen;
}

const char *nombre_tipo(Tipo t) {
    static const char *nombres[] = {"error", "char", "int", "float"};
    return nombres[t];
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Compatibilidad entre tipos, rechazo de TIPO_ERROR y nombres de tipo

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Salida del harness (artesano e IA):** 6/6 pruebas superadas, todas con `OK`.

**Nota sobre accesos fuera de límites:** en mi versión no hay arreglos, así que no hay
riesgo. En la versión de la IA el arreglo de nombres sí se puede salir de límites con
un valor inválido (ver el reporte crítico). El harness no lo prueba, por eso salió en
verde.

**Complejidad:** `compatible` y `nombre_tipo` son O(1) en tiempo y espacio. Solo hacen
unas comparaciones o un acceso directo, sin ciclos ni memoria dinámica.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> La IA pasó los 6 casos del harness, pero omitió casos borde:
> 1. **Acceso fuera de límites en `nombre_tipo`:** usa `nombres[t]` sin revisar el
>    rango. Si llega un valor fuera del enum (por ejemplo `(Tipo)7` o un valor negativo),
>    se lee memoria que no es del arreglo y puede dar basura o un fallo de segmentación.
>    Lo detecté probando a mano `nombre_tipo((Tipo)7)`. Lo solucioné con un `switch`
>    con `default` que devuelve "desconocido", como en mi versión.
> 2. **Valores inválidos en `compatible`:** con un destino fuera del enum, como
>    `compatible((Tipo)7, TIPO_INT)`, la IA devuelve 1 porque 7 >= 2, aceptando un tipo que
>    no existe. Habría que validar que ambos tipos estén en el rango 1 a 3.
> 3. **El orden numérico del enum:** la IA compara los números del enum. Funciona hoy,
>    pero si alguien reordena el enum en el header (por ejemplo agrega TIPO_DOUBLE al
>    principio), la regla de compatibilidad se rompe en silencio. Es mejor una tabla de
>    rangos explícita.
> 4. **Pérdida de precisión de int a float:** la regla acepta int a float como
>    compatible, pero un int grande (más de 2^24) puede perder precisión al volverse
>    float. La especificación lo permite, pero un compilador real debería al menos
>    avisar con una advertencia.
>
> Conclusión: el harness en verde solo comprueba 6 casos válidos y ninguna entrada fuera
> del enum. La IA acertó con la regla principal, pero hay que auditar entradas inválidas
> y depender menos del valor numérico del enum.
---
**Firma digital:** [henari2806-ai]
