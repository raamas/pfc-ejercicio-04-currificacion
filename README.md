# Ejercicio 4 — Currificación

Fundamentos de Programación Funcional y Concurrente
Escuela de Ingeniería de Sistemas y Computación, Universidad del Valle
Carlos Andrés Delgado Saavedra

Cuatro puntos sobre una misma idea: lo que cambia entre varias funciones
parecidas viaja como argumento, y con varios grupos de parámetros se puede
fijar una parte y quedarse con una función que espera el resto.

## Currificar

Currificar es partir una función de varios argumentos en una cadena de
funciones de un argumento. En Scala se escribe con varias listas de
parámetros:

```scala
def suma(x: Int, y: Int): Int = x + y      // una lista, dos parámetros
def suma(x: Int)(y: Int): Int = x + y      // dos listas, una currificada
```

Lo que se gana es la aplicación parcial: si se entregan algunos argumentos y
no todos, lo que queda es una función que espera el resto.

```scala
val sumar5 = suma(5) _     // función que espera un entero
sumar5(3)                  // 8
```

## Lo que hay que resolver

Todo va en `app/src/main/scala/taller/Ejercicio.scala`.

### Punto 1: una operación con cuatro grupos

```scala
def opCurrified(n: Int)(p: Int)(f: (Int, Int) => Int)(g: Int => Int): Int
```

La función recorre `n` términos. El primero es 1 y cada uno se obtiene del
anterior aplicando `g`. Cada término se eleva a la potencia `p`, y los
resultados se combinan con `f`.

| Parámetro | Qué es                                  |
| --------- | --------------------------------------- |
| `n`       | cuántos términos se recorren            |
| `p`       | a qué potencia se eleva cada término    |
| `f`       | cómo se combinan dos resultados         |
| `g`       | cómo se pasa de un término al siguiente |

#### Ejemplos

```scala
opCurrified(3)(2)((x, y) => x + y)(x => x + 1)
```

Los términos son 1, 2 y 3, porque `g` suma uno cada vez. Elevados al
cuadrado: 1, 4 y 9. Combinados con la suma: **14**.

```scala
opCurrified(3)(3)((x, y) => x + y)(x => x + 1)
```

Mismos términos, elevados al cubo: 1, 8 y 27. Suman **36**.

```scala
opCurrified(3)(1)((x, y) => x + y)(x => x + 2)
```

Ahora `g` suma dos, así que los términos son 1, 3 y 5. Con potencia 1 quedan
igual y suman **9**.

```scala
opCurrified(10)(1)((x, y) => x + y)(x => x + 2)
```

Los diez primeros impares: 1, 3, 5, …, 19. Suman **100**.

```scala
opCurrified(1)(4)((x, y) => x + y)(x => x + 1)
```

Un solo término, el 1, elevado a la cuarta. Como no hay con quién combinarlo,
el resultado es **1**. Este es el caso base y conviene escribirlo primero:
con un solo término, `f` no llega a usarse.

Una advertencia sobre el caso base: la función no recibe un elemento
neutro, así que el caso base no puede devolver 0. Si lo hiciera, un `f` de
multiplicación daría siempre cero. El caso de un término tiene que devolver
ese término, y una de las pruebas lo comprueba con la multiplicación:
`opCurrified(4)(1)((x, y) => x * y)(x => x + 1)` es 1 · 2 · 3 · 4 = **24**.

### Punto 2: `suma4`, la suma de la sesión currificada

```scala
def suma4(f: Int => Int)(prox: Int => Int)(a: Int, b: Int): Int
```

Es `suma(f, prox, a, b)` con los parámetros repartidos en tres grupos:
suma `f(a) + f(prox(a)) + …` mientras `a <= b`, y devuelve 0 con un rango
vacío. La llamada recursiva también lleva los tres grupos.

| Llamada                                | Resultado |
| -------------------------------------- | --------- |
| `suma4(x => x)(x => x + 1)(1, 10)`     | 55        |
| `suma4(x => x * x)(x => x + 1)(1, 10)` | 385       |
| `suma4(x => x)(x => x + 2)(1, 10)`     | 25        |
| `suma4(x => x)(x => x * 2)(1, 16)`     | 31        |
| `suma4(x => x)(x => x + 1)(10, 1)`     | 0         |

Con los dos primeros grupos llenos queda una función que solo espera el
rango. `sumaCuadradosSuc` es eso:

```scala
def sumaCuadradosSuc: (Int, Int) => Int
```

`sumaCuadradosSuc(1, 5)` es 55 y `sumaCuadradosSuc(1, 10)` es 385. Se
escribe en una línea, aplicando `suma4` a `x => x * x` y a `x => x + 1` y
sin mencionar `a` ni `b`.

### Punto 3: `reducirC`, de la que `suma` y `producto` son casos

```scala
def reducirC(op: (Int, Int) => Int)(inicio: Int)
            (f: Int => Int, prox: Int => Int)
            (a: Int, b: Int): Int
```

Combina los términos `f(a), f(prox(a)), …` con `op`, y con el rango vacío
devuelve `inicio`. Los dos primeros grupos van juntos porque `inicio` es el
neutro de `op`: 0 para la suma, 1 para el producto.

| Llamada                                                                            | Resultado |
| ---------------------------------------------------------------------------------- | --------- |
| `reducirC((x, y) => x + y)(0)(x => x, x => x + 1)(1, 4)`                           | 10        |
| `reducirC((x, y) => x * y)(1)(x => x, x => x + 1)(1, 4)`                           | 24        |
| `reducirC((x, y) => x + y)(0)(x => x * x, x => x + 2)(1, 7)`                       | 84        |
| `reducirC((x, y) => math.max(x, y))(Int.MinValue)(x => 10 - x, x => x + 4)(1, 13)` | 9         |
| `reducirC((x, y) => x * y)(1)(x => x, x => x + 1)(5, 4)`                           | 1         |

El cuarto caso muestra que `op` no tiene que ser aritmética: el máximo con
`Int.MinValue` como inicio también es una reducción.

Con `reducirC` escritas, `producto` y `factorialHOF` son una línea cada una:

```scala
def producto(f: Int => Int, prox: Int => Int, a: Int, b: Int): Int
def factorialHOF(n: Int): Int
```

`producto(x => x, x => x + 1, 1, 5)` es 120, `producto(x => x * x, x => x + 1, 1, 3)`
es 36, `factorialHOF(0)` es 1 y `factorialHOF(5)` es 120. `factorialHOF` se
escribe en términos de `producto`, y `producto` en términos de `reducirC`.

### Punto 4: funciones que devuelven funciones

```scala
def componer(f: Int => Int)(g: Int => Int): Int => Int
def aplicarN(f: Int => Int)(n: Int): Int => Int
def sumador(n: Int): Int => Int
```

`componer(f)(g)` es la función `x => f(g(x))`: primero `g`, después `f`.
`aplicarN(f)(n)` es la función que aplica `f` n veces; con `n = 0` es la
identidad, y se escribe con `componer` y recursión sobre `n`. `sumador(n)`
es la función que suma `n`.

| Llamada                                 | Resultado |
| --------------------------------------- | --------- |
| `componer(x => x + 1)(x => x * 2)(5)`   | 11        |
| `componer(x => x * 2)(x => x + 1)(5)`   | 12        |
| `aplicarN(x => x * 2)(3)(1)`            | 8         |
| `aplicarN(x => x + 3)(0)(7)`            | 7         |
| `aplicarN(x => x * x)(2)(3)`            | 81        |
| `sumador(5)(3)`                         | 8         |
| `aplicarN(sumador(3))(4)(0)`            | 12        |
| `componer(sumador(5))(sumador(-5))(42)` | 42        |

Los dos primeros dan distinto: el orden de composición importa. Los dos
últimos combinan los tres puntos: `sumador(3)` fabrica una función, y esa
función es lo que `aplicarN` y `componer` reciben.

## Cómo está organizado el proyecto

```
app/src/main/scala/taller/
    App.scala          programa de arranque
    Ejercicio.scala    aquí van los cuatro puntos

app/src/test/scala/taller/
    AppSuite.scala        comprueba que el entorno quedó bien
    EjercicioTest.scala   los casos de las cuatro tablas
```

Su código va en `main`. Las pruebas viven aparte y no se tocan.

## Cómo se ejecuta

```bash
./gradlew test    # corre las pruebas
```

Las pruebas arrancan en rojo y el trabajo es ponerlas en verde. El informe
completo queda en `app/build/reports/tests/test/index.html`.

## Cómo se trabaja

1. Haga fork de este repositorio.
2. En su fork, abra la pestaña **Actions** y habilítelas. GitHub las deja
   desactivadas en las copias hasta que el dueño lo confirme.
3. Clone, resuelva, haga commit y suba a `main`.
4. Verifique en **Actions** que la última ejecución quedó en verde.

## Restricciones

Este curso trabaja sin estado mutable: nada de `var`, `while`, `return` ni
variables que cambien. El resultado correcto por el camino equivocado no
cuenta como resultado correcto.
