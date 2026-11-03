# M04 · ¿Este título va a la portada del usuario?

**Integrantes:** Grosz Luciana · Casabal Guadalupe
**IA declarada:** Claude
**Pista:** Reglas
**Publicación en LinkedIn:** pendiente (se publica en el día; link al repositorio)

---

## 1. Formulación y PEAS

**La pregunta.** Para cada caso (un par usuario–título candidato), ¿el título va a la portada del usuario? **SÍ** significa que el usuario mira **más del 70 % de la duración del título** (por ejemplo, más de 70 minutos de una película de 100). **NO** significa que mira el 70 % o menos.

| PEAS | Nuestro sistema |
|---|---|
| Medida de rendimiento | Que los títulos que pone en portada sean los que el usuario efectivamente mira (más del 70 %), y que no deje afuera los que sí miraría: precisión y sensibilidad, no clics. |
| Entorno | La fila principal de la portada de una plataforma de streaming, con un catálogo de títulos candidatos y el historial de cada usuario. |
| Actuadores | Poner o no poner el título en la fila principal de ese usuario. |
| Sensores | Recibe cuatro datos de cada caso: **afinidad con el historial** (0 a 1; cuánto se parece el título a lo que el usuario ya miró), **duración** (en minutos), **calificación del título** (0 a 10; la nota que tiene en la plataforma) y **clics esperados en la miniatura** (% estimado de gente que haría clic en la imagen). Usa los tres primeros y no usa los clics (ver "Objetivo y métrica"). |

**Objetivo y métrica.** Lo que de verdad importa es que el usuario **termine** lo que le recomendamos (que mire más del 70 %). El equipo de producto, en cambio, mide el éxito con los **clics** en la portada. **No es lo mismo**: alguien puede hacer clic en una miniatura y dejar el título a los pocos minutos. En nuestros datos pasa justo eso: los títulos con más clics esperados son los que menos se terminan (promedian 20 % de clics cuando el usuario no los termina y 10 % cuando sí). Por eso no usamos los clics y medimos con la matriz de confusión (precisión, sensibilidad y especificidad).

---

## 2. El sistema

**Cómo decide.** Un sistema de reglas si‑entonces, que se evalúan en orden y se frena en la primera que se cumple:

1. **Regla 1.** Si la **calificación es 7 o más**, entonces **SÍ**.
2. **Regla 2.** Si la **afinidad es 0,5 o más** **y** la **duración es 100 minutos o menos**, entonces **SÍ**.
3. **Regla 3 (por defecto).** Si no se cumplió ninguna de las anteriores, entonces **NO**.

Los umbrales se eligieron mirando **solo los casos 1 a 14** (entrenamiento). Los casos 15 a 20 se usaron únicamente para medir.
- La Regla 1 usa 7 porque la calificación más alta entre los NO de entrenamiento es 6,4 y la más baja entre los SÍ que entran por esta regla es 7,3.
- La Regla 2 existe porque el caso 6 (afinidad 0,64, 69 min, calificación 5,0) es SÍ y la Regla 1 no lo agarra. Entre los NO de entrenamiento con calificación menor a 7, los de afinidad alta (0,57 y 0,71) duran 169 y 158 minutos, y el único que dura 100 o menos tiene afinidad 0,19; por eso la regla pide afinidad alta **y** duración corta juntas.

**Variables que usa y por qué.**
- **Calificación (Regla 1):** un título muy bien calificado se termina.
- **Afinidad (Regla 2):** cuanto más se parece a lo que el usuario ya mira, más probable que lo termine.
- **Duración (Regla 2):** un título largo se termina menos, aunque tenga afinidad.
- **Descarta: clics esperados en la miniatura.** Está disponible al decidir, pero es la métrica que premia el equipo de producto y no el objetivo. Usarla sería optimizar un indicador que no es lo que importa, y en los datos de entrenamiento va al revés del objetivo.

**Cómo se corre en vivo.** Se hace a mano con los datos del caso nuevo (afinidad, duración, calificación):
1. ¿La calificación es 7 o más? Si sí, **SÍ** y se termina.
2. Si no, ¿la afinidad es 0,5 o más **y** la duración es 100 min o menos? Si sí, **SÍ**.
3. Si no se cumplió ninguna, **NO**.

**Dos casos de ejemplo.**

| Caso inventado | Valores (afinidad / duración / calificación) | Salida del sistema |
|---|---|---|
| A | 0,70 / 80 min / 5,5 | La calificación 5,5 es menor a 7, así que no entra por la Regla 1. Afinidad 0,70 ≥ 0,5 y duración 80 ≤ 100: entra por la **Regla 2** → **SÍ** |
| B | 0,20 / 150 min / 4,5 | Calificación menor a 7; afinidad menor a 0,5 y duración mayor a 100. No entra por ninguna: **Regla 3** → **NO** |

---

## 3. Medición

**Entrenamiento (casos 1–14)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 7 | FN = 0 |
| **Real NO** | FP = 0 | VN = 7 |

**Prueba (casos 15–20)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 2 | FN = 0 |
| **Real NO** | FP = 2 | VN = 2 |

| Métrica | Entrenamiento | Prueba |
|---|---|---|
| Exactitud | 14/14 = 100 % | 4/6 = 66,7 % |
| Precisión | 7/7 = 100 % | 2/4 = 50 % |
| Sensibilidad | 7/7 = 100 % | 2/2 = 100 % |
| Especificidad | 7/7 = 100 % | 2/4 = 50 % |

**Entrenamiento contra prueba.** Hay una diferencia grande: 100 % contra 66,7 % de exactitud. Es esperable porque armamos las reglas a mano mirando los 14 casos de entrenamiento, así que quedaron pegadas a ellos (posible sobreajuste), y porque la prueba tiene solo 6 casos: uno solo cambia 16,7 puntos. Los dos errores de prueba son falsos positivos y muestran el límite de cada regla:
- **Caso 18** (afinidad 0,34, 116 min, calificación 9,3): la Regla 1 mira solo la calificación y lo manda a portada, aunque la afinidad con ese usuario es baja.
- **Caso 20** (afinidad 0,71, 55 min, calificación 3,9): la Regla 2 mira solo afinidad y duración e ignora que la calificación es baja.

No cambiamos las reglas después de ver la prueba, porque esos casos son solo para medir.

---

## 4. El error más caro

**El falso positivo es más caro.** Según la ficha, poner en portada algo que el usuario no termina desperdicia un lugar y además hace que sienta que la plataforma no lo conoce. La fila principal es un espacio escaso: cada lugar ocupado por un título que no se mira es un lugar que no tiene otro título que sí se habría mirado. El falso negativo (dejar afuera algo que habría disfrutado) es una oportunidad perdida, pero el catálogo tiene muchos otros candidatos que pueden ocupar ese lugar.

**Qué hicimos en el sistema por eso.** Está armado para decir SÍ solo con evidencia fuerte:
- La regla por defecto es **NO**: ante la duda, el título no va a portada.
- La Regla 2 pide dos condiciones juntas (afinidad alta **y** duración corta), no una sola.
- El umbral de la Regla 1 (7) queda por encima de la calificación más alta de los NO de entrenamiento (6,4).

Aun así, en prueba cometió 2 falsos positivos (casos 18 y 20). Anotamos que cada regla ignora una de las variables; no lo corregimos mirando la prueba y queda como punto a revisar con lo que marque el evaluador.

---

## 5. ¿Comparado contra qué?

**Línea base trivial.** Un sistema que responde siempre lo mismo (la clase más frecuente):
- En entrenamiento hay 7 SÍ y 7 NO, así que acierta 7/14 = **50 %**. Nuestro sistema saca 100 %, y lo supera por mucho.
- En prueba hay 2 SÍ y 4 NO, así que "siempre NO" acierta 4/6 = **66,7 %**. Nuestro sistema saca 66,7 %: **empata con la línea base** en exactitud. Con tan pocos casos no podemos afirmar que sea mejor que responder siempre NO en datos nuevos. Lo que sí hace es detectar los dos SÍ (sensibilidad 100 %), cosa que "siempre NO" nunca hace (0 %), a costa de 2 falsos positivos.

| Checklist (Clase 6) | Respuesta |
|---|---|
| ¿Qué datos? | 20 casos inventados de la ficha M04 (14 de entrenamiento, 6 de prueba). No hay datos reales de personas. |
| ¿Qué métrica? | Matriz de confusión con exactitud, precisión, sensibilidad y especificidad. No usamos clics, que es la métrica del equipo de producto. |
| ¿Comparado contra qué? | Contra la línea base de responder siempre la clase más frecuente (50 % en entrenamiento, 66,7 % en prueba). |
| ¿Quién lo midió? | Nosotras mismas, con los datos de la ficha y sin validación externa. Se suma la corrida contra el evaluador (ver bitácora). |

---

## 6. Riesgo: el camino completo

**Datos → modelo → decisión → consecuencia.**
- **Datos:** solo 20 casos inventados, 14 para armar las reglas. Es una muestra muy chica.
- **Modelo:** dos reglas con umbrales (7, 0,5 y 100 min) puestos a mano sobre esos 14 casos, que sacaron 100 % en entrenamiento y 66,7 % en prueba (sobreajuste probable). Cada regla ignora al menos una variable.
- **Decisión:** el recomendador pone o no el título en la portada según qué regla se cumple.
- **Consecuencia:** un título muy bien calificado pero que no tiene nada que ver con el usuario (como el caso 18) llega igual a su portada, y el usuario siente que la plataforma no lo conoce. Hay un segundo riesgo: si producto sigue premiando los clics, podrían empujar al sistema a usar la columna de clics, que en nuestros datos va al revés del objetivo, y la portada terminaría llena de títulos muy clickeables que nadie termina de ver.

---

## 7. Bitácora

**Uso de la IA**

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
|---|---|---|---|
| 1 | Que leyera las bases y la ficha M04 y armara un sistema usando solo los casos 1 a 14, midiendo después en los casos 15 a 20. | Las tres reglas (calificación ≥ 7; afinidad ≥ 0,5 y duración ≤ 100; si no, NO), las matrices y las métricas. | Usar "clics esperados" como variable: está disponible, pero es la métrica del equipo de producto y no el objetivo, y en entrenamiento va al revés (promedio 10 en los SÍ contra 20 en los NO). Además, la IA había propuesto primero una neurona; el grupo eligió la pista Reglas. |
| 2 | Que comparara el resultado contra la línea base. | La comparación: en prueba la exactitud empata con "siempre NO" (66,7 %). | Retocar las reglas mirando los casos de prueba para mejorar el número: las bases dicen que esos casos son solo para medir. |

**Corridas contra el evaluador**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | | | |
