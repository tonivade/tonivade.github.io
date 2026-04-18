---
layout: post
title:  Programando con DSLs en Java 25 (VI)
date:   2026-04-18 14:00:00
categories: programming java functional-programming dsl monad
---

Como buen guionista de Hollywood, siempre hay que dejar abierto los finales de 
las películas para poder hacer otra parte, si la película anterior tiene éxito.
No es que haya tenido mucho éxito con mis artículos, pero mientras a mi me
siga divirtiendo escribirlos, no necesito más escusa. Asi que amenazo con
continuar escribiendo sobre esto, no solo hoy, sino en un futuro cercano.

Si habéis estado atentos a los otro artículos, supongo que os habrá llamado la
atención una cosa, y es que la implementación del método `eval` era recursiva.
Volvamos a ver el código de la última versión del método:

```java
@SuppressWarnings("unchecked")
default Either<E, T> eval(S state) {
  return switch (this) {
    case Success<S, E, T>(T value) -> Either.right(value);
    case Failure<S, E, T>(E error) -> Either.left(error);
    case FlatMap<S, E, ?, T>(var current, var next) -> {
    var result = current.eval(state); // <- recursion here
    yield result.fold(
      _ -> (Either<E, T>) result,
      value -> {
        var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(value);
        return nextValue.eval(state);// <- recursion here
      });
    }
    case FlatMapError<S, ?, E, T>(var current, var next) -> {
    var result = current.eval(state); // <- recursion here
    yield result.fold(
      error -> {
        var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(error);
        return nextValue.eval(state); // <- recursion here
      },
      _ -> (Either<E, T>) result);
    }
    case Access<S, E, T>(var mapper) -> Either.right(mapper.apply(state));
  };
}
```

Para el ejemplo con el que estábamos trabajando esta implementación es perfectamente
válida. Además aquí usamos muchas de las nuevas mejoras del lenguaje Java, pattern
matching, improved switch expressions, etc. Pero si tuviéramos un programa más complejo
que fuera a su vez fuera recursivo, pues tendríamos un problema.

Por ejemplo, si queremos implementar una función muy sencilla que simplemente sume todos
número sucesivos desde el uno hasta el número que queramos. Podríamos hacerlo de esta
manera usando nuestra librería:

```java
static Program<Void, Void, Integer> recursiveSum(int n, int sum) {
  if (n == 0) {
    return success(sum);
  }
  return pipe(unit(), _ -> recursiveSum(n - 1, n + sum));
}
```

Como buena función recursiva, tiene un caso base que rompe la recursividad, cuando
`n` vale `0`. Y el caso donde se entra en recursividad. Como se puede ver,
el método recibe dos parámetros, en número actual y el resultado de sumar los
números anteriores. Y cuando `n` es '0` simplemente devolvemos la suma de los
números anteriores como resultado.

Probemos, a ver qué pasa si ejecutamos esto:

```java
@Test
void shouldBeStackSafe() {
  var program = recursiveSum(100_000, 0);

  var result = program.eval(null);

  assertEquals(Either.right(705_082_704), result);
}
```

El resultado es este, un bonito desbordamiento de pila:

```
java.lang.StackOverflowError
	at ProgramTest.lambda$0(ProgramTest.java:25)
	at Program.lambda$4(Program.java:86)
	at Either.fold(Either.java:13)
	at Program.eval(Program.java:83)
	at Program.lambda$4(Program.java:87)
	at Either.fold(Either.java:13)
	at Program.eval(Program.java:83)
	at Program.lambda$4(Program.java:87)
  ...
```

Otros lenguajes de programación tienen algo llamado [tail-call optimization](https://en.wikipedia.org/wiki/Tail_call)
que permiten escribir programas recursivos que estén libres de problemas
de desbordamiento de pila.

Imaginemos que tenemos una función parecida pero escrita de manera imperativa:

```java
static int unsafeSum(int n, int sum) {
  if (n == 0) {
    return sum;
  }
  return unsafeSum(n - 1, n + sum);
}
```

Esta función tiene el mismo problema de desbordamiento de pila que nuestra versión 
declarativa. ¿Qué pasa al ejecutar esto? Hagamos un esquema para verlo más claro.
Cada vez que incrementamos la identación entramos en un nivel más en la pila. En
algún momento con un número `n` lo suficientemente algo, la pila se desborda.

```
  unsafeSum(10, 0)
  \-> unsafeSum(9, 10)
     \-> unsafeSum(8, 19)
        \-> ...
```

Pues con tail-call optimization, el compilador detecta que la función se llama
a si misma justo al final, lo que le permite optimizarla. Esto nos permite aplanar 
el diagrama, ya que en lugar de entrar en otro nivel de la pila, es como si
hiciéramos un go-to al inicio principio de la función.

```
  unsafeSum(10, 0)
  unsafeSum(9, 10)
  unsafeSum(8, 19)
  ...
```

De esta forma nuestra función estaría libre de desbordamientos de pila. Lamentablemente
en Java esto no existe. Otros lenguajes de programación como Scala o Kotlin si lo
permiten.

¿Qué podemos hacer? Hay una estructura llamada trampolín que nos permitiría hacer
programas recursivos libres de desbordamientos de pila. La implementación es muy
sencilla:

```java
sealed interface Trampoline<T> {
  record Done<T>(T value) implements Trampoline<T> {}
  record More<T>(Supplier<Trampoline<T>> next) implements Trampoline<T> {}
}
```

Y nos faltaría implementar el bucle de evaluación, queremos que esté libre
de problemas de desbordamiento de pila por lo que no debemos implementarlo
de manera recursiva, como hicimos para `Program`, sino iterativa.

```java
  default T eval() {
    Trampoline<?> current = this;

    while (true) {
      if (current instanceof Done(var value)) {
        return (T) value; // end of program
      } else if (current instanceof More(var next)) {
        current = next.get();
      }
    }
  }
```

Ahora podríamos implementar nuestro programa de sumas usando `Trampoline`:

```java
static Trampoline<Integer> recursiveSum(int n, int sum) {
  if (n == 0) {
    return new Done<>(sum);
  }
  return new More<>(() -> recursiveSum(n - 1, n + sum));
}
```

Y este programa está libre de desbordamientos de pilas, no importa lo grande
que sea `n`. ¿Por qué funciona? Pues porque con `More` decimos "no ejecutes
esto ahora, ejecútalo más tarde". Es como si suspendiéramos la ejecución en ese
momento. Hasta que no se evalúe, no se irán generando los diferentes programas
intermedios. Son como las muñecas rusas, solo que no sabes si al destaparla
habrá otra muñeca dentro, o si ya has llegado al final.

Con `Trampoline` también podemos escribir programas un poco más complejos, como por ejemplo
el cálculo de la [sucesión de Fibonacci](https://es.wikipedia.org/wiki/Sucesi%C3%B3n_de_Fibonacci).
Esta se calcula de esta manera:

- `fib(0) = 1`
- `fib(1) = 1`
- `fib(2) = fib(0) + fib(1) = 1 + 1 = 2`
- `fib(3) = fib(1) + fib(2) = 1 + 2 = 3`
- `fib(4) = fib(2) + fib(3) = 2 + 3 = 5`

Generalizando tenemos que: `fib(n) = fib(n - 2) + fib(n -1)`

Con la definición actual de `Trampoline` tal y como está, no se podría implementar, pero podemos
fácilmente adaptarlo para hacerlo funcionar. Veamos:

```java
static Trampoline<Integer> fib(int n) {
  if (n == 0) {
    return new Done<>(1);
  }
  if (n == 1) {
    return new Done<>(1);
  }
  return fib(n - 2) + fib(n - 1);
}
```

Los casos `0` y `1` son triviales, simplemente devolvemos `1`. Ahora bien ¿como hacemos para
implementar el caso más general? Necesitamos calcular la secuencia de Fibonacci de `n - 2`,
la secuencia de Fibonacci de `n - 1` y sumarlas. Pero claro, nuestra función `fib` devuelve
un `Trampoline<Integer>`, ¿qué podemos hacer para sumarlos? Al igual que hicimos en `Program` 
necesitamos algo que haga de pegamento y poder enlazar la ejecución de un programa con otro, 
nuestro famoso `AndThen`.

```java
sealed interface Trampoline<T> {
  record Done<T>(T value) implements Trampoline<T> {}
  record More<T>(Supplier<Trampoline<T>> next) implements Trampoline<T> {}
  record AndThen<T, R>(
    Trampoline<T> current, Function<T, Trampoline<R>> mapper) implements Trampoline<R> {}

  default <R> Trampoline<R> andThen(Function<T, Trampoline<R>> next) {
    return new AndThen<>(this, next);
  }
}
```

Añadamos también estas funciones de ayuda:

```java
static <T> Trampoline<T> done(T value) {
  return new Done<>(value);
}

static <T> Trampoline<T> more(Supplier<Trampoline<T>> next) {
  return new More<>(next);
}
```

Ahora podríamos implementar nuestro programa para calcular Fibonacci de esta manera:

```java
static Trampoline<Integer> fib(int n) {
  if (n == 0) {
    return done(1);
  }
  if (n == 1) {
    return done(1);
  }
  var fib2 = more(() -> fib(n - 2));
  var fib1 = more(() -> fib(n - 1));
  return fib2.andThen(i -> fib1.andThen(j -> done(i + j)));
}
```

Primero generamos el programa que genera la secuencia de Fibonacci de `n - 2`,
luego el de `n - 1` y los combinamos usando `andThen`.

¿Cómo implementaríamos `eval` para soportar nuestro `AndThen`, y además
hacerlo de tal manera que esté libre de desbordamientos de pila?

```java
default T eval() {
  Trampoline<?> current = this;

  while (true) {
    if (current instanceof Done(var value)) {
      return (T) value; // end of program
    } else if (current instanceof More(var next)) {
      current = next.get();
    } else if (current instanceof AndThen(var source, var next)) {
      // ???
    }
  }
}
```

`source` es un `Trampoline` y `next` es una función que necesita que `source`
se ejecute para, a partir del resultado, poder calcular el siguiente programa.
Pero claro, `source` todavía no se ha ejecutado, ¿qué es lo que podemos hacer?
Pues hay que dejar la ejecución de `source` para más tarde. Y una vez que tengamos
el valor que produce, poder ejecutar la función `next`. Es decir tenemos que 
simular una especie de pila.

```java
default T eval() {
  Trampoline<?> current = this;
  Deque<Function<Object, ? extends Trampoline<?>>> stack = new ArrayDeque<>();

  while (true) {
    if (current instanceof Done(var value)) {
      if (stack.isEmpty()) {
        return (T) value; // end of program
      }
      current = stack.pop().apply(value);
    } else if (current instanceof More(var next)) {
      current = next.get();
    } else if (current instanceof AndThen(var source, var next)) {
      stack.push((Function<Object, ? extends Trampoline<?>>) next);
      current = source;
    }
  }
}
```

Para ello vamos a usar una estructura de datos que sea como una pila, en nuestro
caso usaremos `Deque`. Encolamos la función `next` en la pila y asignamos
`source` a `current`. Hay que hacer un cast para dejar contento al compilador.

Así en la siguiente iteración del bucle, se evaluará `source` y obtendremos el
valor de ejecutar `source`. Luego podremos ejecutar la función `next` para obtener 
el siguiente programa.

Ahora, en el caso de `Done`, tendremos que ir evaluando las funciones de la pila. 
Si la pila está vacía, significa que ya hemos terminado y simplemente devolvemos el 
valor de `Done`. Si la pila no está vacía, sacamos la primera función del `stack` y 
la ejecutamos pasando el valor de `Done` como parámetro. Con eso obtenemos el siguiente 
programa que se asigna a `current`, que a su vez se evaluará en la siguiente iteración
del bucle.

Puede parecer todo un poco enrevesado, pero tiene mucho sentido.

Ahora cómo aplicamos todo esto en nuestra función `eval` de `Program`, que es de
lo que va todo esto. Simplemente necesitamos generalizar lo que hemos hecho en
`Trampoline`. En el caso de `Program` necesitaremos dos diferentes stacks, uno
para cuando va todo bien y otro para cuando algo va mal. Los casos de `Success`
y `Failure`, son equivalentes a `Done`, la diferencia es que ejecutan las
funciones de una pila u otra. `Failure` de la pila de cuando algo va mal, y 
`Success` de la pila de cuando todo va bien. Quedaría de esta manera:

```java
default Either<E, T> eval(S state) {
  Program<S, ?, ?> current = this;
  Deque<Function<Object, ? extends Program<S, ?, ?>>> failureStack = new ArrayDeque<>();
  Deque<Function<Object, ? extends Program<S, ?, ?>>> successStack = new ArrayDeque<>();

  while (true) {
    if (current instanceof Success(var value)) {
      if (successStack.isEmpty()) {
        return Either.right((T) value);
      }
      current = successStack.poll().apply(value);
    } else if (current instanceof Failure(var error)) {
      if (failureStack.isEmpty()) {
        return Either.left((E) error);
      }
      current = failureStack.poll().apply(error);
    } else if (current instanceof Access(var mapper)) {
      current = mapper.apply(state);
    } else if (current instanceof FlatMap(var next, var mapper)) {
      successStack.push((Function<Object, ? extends Program<S, ?, ?>>) mapper);
      current = next;
    } else if (current instanceof FlatMapError(var next, var mapper)) {
      failureStack.push((Function<Object, ? extends Program<S, ?, ?>>) mapper);
      current = next;
    }
  }
}
```

Ahora nuestro programa original funcionaría sin ningún problema:

```java
static Program<Void, Void, Integer> recursiveSum(int n, int sum) {
  if (n == 0) {
    return success(sum);
  }
  return pipe(unit(), _ -> recursiveSum(n - 1, n + sum));
}
```

Podríamos crear una función de ayuda que llamaríamos `suspend`:

```java
static <S, E, T> Program<S, E, T> suspend(Supplier<Program<S, E, T>> program) {
  return pipe(unit(), _ -> program.get());
}
```

Y simplificar un poco esto:

```java
static Program<Void, Void, Integer> recursiveSum(int n, int sum) {
  if (n == 0) {
    return success(sum);
  }
  return suspend(() -> recursiveSum(n - 1, n + sum));
}
```

Y que pasa con la sucesión de Fibonacci, ¿podemos implementarla con `Program`?
Pues claro que si.

```java
static Program<Void, Void, Integer> fib(int n) {
  if (n == 0) {
    return success(1);
  }
  if (n == 1) {
    return success(1);
  }
  var fib2 = suspend(() -> fib(n - 2));
  var fib1 = suspend(() -> fib(n - 1));
  return pipe(fib2, i -> fib1.map(j -> i + j));
}
```

Y una última cosa, que sino no lo digo exploto, ¿esto no se podría mejorar?

```java
pipe(fib2, i -> fib1.map(j -> i + j))
```

Aquí `pipe` no es lo suficientemente expresivo, `pipe` está pensado para
ejecutar operaciones que dependen una del resultado de la anterior. Aquí
necesitamos otro tipo de combinador.

```java
static <S, E, T, U, R> Program<S, E, R> zip(
    Program<S, E, T> first, Program<S, E, U> second, BiFunction<T, U, R> mapper) {
  return pipe(first, t -> second.map(u -> mapper.apply(t, u)));
}
```

`zip` recibe dos programas como parámetro y una función que combina el resultado
de los dos programas.

```java
static Program<Void, Void, Integer> fib(int n) {
  if (n == 0) {
    return success(1);
  }
  if (n == 1) {
    return success(1);
  }
  var fib2 = suspend(() -> fib(n - 2));
  var fib1 = suspend(() -> fib(n - 1));
  return zip(fib2, fib1, Integer::sum);
}
```

Y eso es todo lo que quería contar, por ahora. Como he dicho al principio, no puedo 
prometer que este sea el último artículo de la serie.