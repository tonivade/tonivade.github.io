---
layout: post
title:  Programando con DSLs en Java 25 (V)
date:   2026-04-03 14:00:00
categories: programming java functional-programming dsl monad
---

Durante estos 4 últimos artículos hemos ido paso a paso hasta implementar un pequeño
framework para desarrollar mi lenguajes y combinarlos para desarrollar programas.

Y hemos visto que, de manera natural, lo que obtenemos es una estructura parecida 
a una free monad, que viene del mundo de la programación funcional.

Hemos llegado al punto que podemos combinar operaciones usando el método `pipe` por
ejemplo:

```java
pipe(
  println("What's your name?"),
  _ -> readln(),
  name -> println("Hello " + name + "!")
);
```

En `pipe` la salida de una operación es la entrada de la siguiente, por lo que tenemos
una especie de pipeline.

Veamos un ejemplo del artículo anterior donde teníamos este caso, se le pedía al usuario
introducir un número entre 0 y 9 y luego la entrada del usuario se convertía en un número
usando `Integer.parseInt`.

```java
static <S extends Console.Service> Program<S, Integer> readNumber() {
  return pipe(
    prompt("Enter a number between 0 to 9"),
    lift(Integer::parseInt)
  );
}
```

¿Qué ocurriría si el usuario introduce un valor que no se pueda convertir a un número
entero? Pues que se lanzaría una excepción en runtime.

```
Do you want to play a game? (y/n)
y
Enter a number between 0 to 9
c
Exception in thread "main" java.lang.NumberFormatException: For input string: "c"
	at java.base/java.lang.NumberFormatException.forInputString(NumberFormatException.java:67)
```

Esto nos lleva a preguntarnos cómo podríamos controlar estos errores dentro de nuestro
mini lenguaje. Podríamos simplemente capturar la excepción y tratar de recuperarnos
del error:

```java
static <S extends Console.Service> Program<S, Integer> readNumber() {
  return pipe(
    prompt("Enter a number between 0 to 9"),
    s -> {
      try {
        return new Program.Done<>(Integer.parseInt(s));
      } catch (NumberFormatException e) {
        return pipe(
            println("Invalid value"),
            _ -> readNumber());
      }
    }
  );
}
```

En este código parseamos la entrada, si todo va bien devolvemos el resultado wrappeadolo
en un objeto `Program.Done`, y en caso de que se produzca una excepción simplemente
se imprime por pantalla un error y volvemos a intentarlo llamando de nuevo al mismo
método `readNumber`.

El método `readNumber` se usa dentro del programa de esta manera.

```java
static <S extends Console.Service & State.Service> Program<S, Exception, Void> play() {
  return pipe(
    readNumber(),
    Game::checkNumber,
    result -> {
      if (result) {
        return println("YOU WIN!");
      }
      return play();
    }
  );
}
```

No está mal, ahora podemos capturar el error y volver a preguntar al usuario hasta 
que introduzca un valor correcto. Pero ¿no podríamos generalizar la gestión de errores, 
y que esta esté integrada en nuestro propio mini lenguaje?

En lenguajes como Java tenemos las excepciones para gestionar errores. Las excepciones
rompen el flujo normal de ejecución, por lo que siendo puristas se pueden considerar
side effects, lo mismo que leer o escribir de la salida estándar. 

Pero hay otras maneras de gestionar errores, podemos hacer que un método responda con
una estructura de datos que cuando funciona devuelva un tipo de dato, y cuando falla otro
diferente. Esto se suele representar utilizando con el tipo `Either`. En Java podemos 
implementarlo usando sealed interfaces, algo como esto:

```java
sealed interface Either<L, R> {

  record Left<L, R>(L left) implements Either<L, R> {}
  record Right<L, R>(R right) implements Either<L, R> {}
}
```

Entonces nuestro método `eval` en `Program` ya no puede devolver simplemente `T`, sino un
`Either`. Por lo general, el tipo de dato a la derecha (right) suele representar el tipo
de dato cuando todo se ejecuta correctamente, y el tipo de dato a la izquierda (left) el
tipo de dato del error. Por lo tanto devolveremos `Either<?, T>`.

```java
@SuppressWarnings("unchecked")
default Either<?, T> eval(S state) {
  return switch (this) {
    // ...
  };
}
```

Pero en el caso de error el tipo de error no puede ser simplemente `?`, necesitamos
un tipo de dato ahí. Asi que debemos definir algo como esto:

```java
@SuppressWarnings("unchecked")
default Either<E, T> eval(S state) {
  return switch (this) {
    // ...
  };
}
```

Ahora `eval` devuelve `Either<E, T>`, ¿pero `E` de donde sale? Necesitamos añadir en
`Program` otro parámetro para definir el tipo de error que puede devolver el programa.
Y por lo tanto, ese `E` se debe propagar a todas las implementaciones de `Program`.

```java
sealed interface Program<S, E, T> {

  record Done<S, E, T>(T value) implements Program<S, E, T> {}

  record FlatMap<S, E, X, T>(Program<S, E, X> current, Function<X, Program<S, E, T>> next) implements Program<S, E, T> {}

  record Access<S, E, T>(Function<S, T> mapper) implements Program<S, E, T> {}
}
```

Entonces ahora, cuando falla un programa ¿qué devuelve? `Done` no nos sirve ya que se
corresponde al caso cuando todo va bien. Por lo que necesitaremos otra implementación de
`Program` para los casos cuando falla.

```java
sealed interface Program<S, E, T> {

  record Success<S, E, T>(T value) implements Program<S, E, T> {}

  record Failure<S, E, T>(E error) implements Program<S, E, T> {}

  record FlatMap<S, E, X, T>(Program<S, E, X> current, Function<X, Program<S, E, T>> next) implements Program<S, E, T> {}

  record Access<S, E, T>(Function<S, T> mapper) implements Program<S, E, T> {}
}
```

`Done` la hemos renombrado como `Success` y hemos definido el nuevo tipo como `Failure`.

Ahora bien, cómo queda el método `eval`:

```java
@SuppressWarnings("unchecked")
default Either<E, T> eval(S state) {
  return switch (this) {
    case Success<S, E, T>(T value) -> new Either.Right<>(value);
    case Failure<S, E, T>(E error) -> new Either.Left<>(error);
    // ...
  };
}
```

Volvamos al ejemplo anterior pero ahora teniendo en cuenta nuestros dos tipos
nuevos: `Success` y `Failure`.

```java
static <S> Program<S, NumberFormatException, Integer> parseInt(String s) {
  try {
    return new Program.Success<>(Integer.parseInt(s));
  } catch (NumberFormatException e) {
    return new Program.Failure<>(e);
  }
}
```

Este nuevo método gestiona el error de `Integer.parseInt` y devuelve `Failure` cuando
se produce una excepción, y `Success` cuando va todo bien. Esto nos permite
capturar el error, pero todavía no tenemos posibilidad de gestionarlos, para ello
necesitaremos otro tipo de dato que reciba el error y devuelva un programa. Algo
parecido a lo que ya hace `FlatMap`, pero para errores. Algo como esto:

```java
record FlatMapError<S, X, E, T>(Program<S, X, T> current, Function<X, Program<S, E, T>> next) 
    implements Program<S, E, T> {}
```

Ahora necesitamos adaptar el método `eval` para gestionar el nuevo tipo `FlatMapError`:

```java
@SuppressWarnings("unchecked")
default Either<E, T> eval(S state) {
  return switch (this) {
    case Success<S, E, T>(T value) -> Either.right(value);
    case Failure<S, E, T>(E error) -> Either.left(error);
    case FlatMap<S, E, ?, T>(var current, var next) -> {
      var result = current.eval(state);
      yield result.fold(
          _ -> (Either<E, T>) result,
          value -> {
            var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(value);
            return nextValue.eval(state);
          });
    }
    case FlatMapError<S, ?, E, T>(var current, var next) -> {
      var result = current.eval(state);
      yield result.fold(
          error -> {
            var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(error);
            return nextValue.eval(state);
          },
          _ -> (Either<E, T>) result);
    }
    case Access<S, E, T>(var mapper) -> Either.right(mapper.apply(state));
  };
}
```

Aquí vemos muchas cosas nuevas, las implementaciones de `FlatMap` y `FlatMapError` llaman
a `eval` por lo que el resultado de llamar a `current.eval()` es un tipo `Either`. Luego se llama 
al método `fold` de `Either`, que permite transformar `Either`. `fold` recibe dos funcionas, 
una para el gestionar el caso left y otra para gestionar el caso right.

```java
default <T> T fold(Function<L, T> leftCase, Function<R, T> rightCase) {
  return switch (this) {
    case Left<L, R>(var left) -> leftCase.apply(left);
    case Right<L, R>(var right) -> rightCase.apply(right);
  };
}
```

Ahora nos queda definir un método similar a `flatMap` pero para gestionar errores:

```java
default <F> Program<S, F, T> flatMapError(Function<E, Program<S, F, T>> recover) {
  return new FlatMapError<>(this, recover);
}
```

Volviendo al ejemplo anterior, el método `readNumber` usaría el método `parseInt` que hemos
implementado antes que captura el error. Y llamamos a `flatMapError` para recuperarnos
del error:

```java
static <S extends Console.Service> Program<S, NumberFormatException, Integer> readNumber() {
  return pipe(
    prompt("Enter a number between 0 to 9"),
    Game::parseInt
  ).flatMapError(_ -> pipe(println("Invalid value"), _ -> readNumber()));
}
```

Aquí volvemos a tener el mismo problema que con `flatMap`, ya que el compilador no es capaz
de inferir correctamente los tipos. Para resolverlo podemos hacer algo parecido a lo que 
hicimos con el método `pipe` en el artículo anterior:

```java
static <S, E, F, T> Program<S, F, T> recover(
    Program<S, E, T> first, Function<E, Program<S, F, T>> next) {
  return first.flatMapError(next);
}
```

Esto sería algo parecido a un try-catch pero de una manera declarativa.

Ahora quedaría nuestro código de esta manera:

```java
static <S extends Console.Service> Program<S, Exception, Integer> readNumber() {
  return recover(
    pipe(
      prompt("Enter a number between 0 to 9"),
      Game::parseInt
    ),
    _ -> pipe(println("Invalid value"), _ -> readNumber())
  );
}
```

Ya por fin el compilador es capaz de inferir los tipos correctamente.

Ahora si ejecutamos el programa esto es lo que veríamos:

```
Do you want to play a game? (Y/y)
y
Enter a number between 0 to 9
c
Invalid value
Enter a number between 0 to 9
2
Enter a number between 0 to 9
3
YOU WIN!!
```

Como vemos, ahora somos capaces de capturar y gestionar los errores dentro de nuestros
programas. Y para estar seguros vamos hacer un test unitario, similar a los que 
implementamos en el artículo anterior.

```java
@Test
void userEnterAnInvalidValue() {
  var context = new MockContext("y", "c", "2");

  game().eval(context);

  var expectedOutput = List.of(
      "Do you want to play a game? (y/n)",
      "Enter a number between 0 to 9",
      "Invalid value",
      "Enter a number between 0 to 9",
      "YOU WIN!");
  assertEquals(expectedOutput, context.output);
}
```

Una última cosa que podemos hacer es definir una función de ayuda para generalizar
el caso del método `parseInt` que hemos hecho antes.

```java
static <S> Program<S, NumberFormatException, Integer> parseInt(String s) {
  try {
    return new Program.Success<>(Integer.parseInt(s));
  } catch (NumberFormatException e) {
    return new Program.Failure<>(e);
  }
}
```

Hay un patron bastante claro, cuando va bien devuelve `Success` cuando hay una excepción
devuelve `Failure`. Podríamos hacer algo como esto:

```java
static <S, T> Program<S, Exception, T> attempt(Supplier<T> supplier) {
  try {
    return success(supplier.get());
  } catch (Exception e) {
    return failure(e);
  }
}
```

Esto quedaría en nuestro código de esta manera:

```java
static <S extends Console.Service> Program<S, Exception, Integer> readNumber() {
  return recover(
      pipe(
        prompt("Enter a number between 0 to 9"),
        s -> attempt(() -> Integer.parseInt(s))
      ),
    _ -> pipe(println("Invalid value"), _ -> readNumber()));
}
```

Y aquí termina nuestro viaje, empezamos implementando unos ejemplos muy sencillos,
primero solo necesitando combinar operaciones, luego combinar operaciones que
existen en un contexto, y finalmente capturar y gestionar errores. Hemos terminado 
implementando un mini lenguaje que nos permite hacer todas esas cosas.

Y eso es exactamente lo que he implementado en mi libraría [diesel](https://github.com/tonivade/diesel).
Además de implementar todo esto que he explicado en estos artículos y más, hay un
procesador de anotaciones que automáticamente nos genera las diferentes operaciones
definidas en un interfaz, en operaciones que podemos combinar en un DSL. 

Por ejemplo, imaginemos que tenemos este interfaz:

```java
import com.github.tonivade.diesel.Diesel;

@Diesel
public interface Console {
  String readLine();
  void writeLine(String line);
}
```

Con la anotación `@Diesel` el procesador de anotaciones generará el siguiente código:

```java
import com.github.tonivade.diesel.Program;
import javax.annotation.processing.Generated;

@Generated("com.github.tonivade.diesel.DieselAnnotationProcessor")
public interface ConsoleDsl {

  static <S extends Console, E> Program<S, E, String> readLine() {
    return Program.effect(Console::readLine);
  }

  static <S extends Console, E> Program<S, E, Void> writeLine(String line) {
    return Program.effect(console -> {
      console.writeLine(line);
      return null;
    });
  }
}
```

Y ya podremos usarlo en nuestros programas:

```java
import static ConsoleDsl.*;
import static com.github.tonivade.diesel.Program.*;

public static void main(String... args) {

  var program = pipe(
    writeLine("What's your name?"),
    _ -> readLine(),
    name -> writeLine("Hello " + name + "!")
  );

  // output of the program:
  // >> What's your name?
  // << Toni
  // >> Hello Toni!
  program.eval(new Console() {
    public void writeLine(String line) {
      IO.println(line);
    }
    public String readLine() {
      return IO.readln();
    }
  });
}
```

Y por último dejo aquí el código completo del ejemplo:

```java
import java.util.function.Function;
import java.util.function.Supplier;

sealed interface Program<S, E, T> {

  record Success<S, E, T>(T value) implements Program<S, E, T> {}

  record Failure<S, E, T>(E error) implements Program<S, E, T> {}

  record FlatMap<S, E, X, T>(Program<S, E, X> current, Function<X, Program<S, E, T>> next) implements Program<S, E, T> {}

  record FlatMapError<S, X, E, T>(Program<S, X, T> current, Function<X, Program<S, E, T>> next) implements Program<S, E, T> {}

  record Access<S, E, T>(Function<S, T> mapper) implements Program<S, E, T> {}

  default <R> Program<S, E, R> map(Function<T, R> mapper) {
    return flatMap(lift(mapper));
  }

  default <R> Program<S, E, R> flatMap(Function<T, Program<S, E, R>> next) {
    return new FlatMap<>(this, next);
  }

  default <F> Program<S, F, T> flatMapError(Function<E, Program<S, F, T>> recover) {
    return new FlatMapError<>(this, recover);
  }

  static <S, E, T> Program<S, E, T> success(T value) {
    return new Success<>(value);
  }

  static <S, E, T> Program<S, E, T> failure(E error) {
    return new Failure<>(error);
  }

  static <S, T> Program<S, Exception, T> attempt(Supplier<T> supplier) {
    try {
      return success(supplier.get());
    } catch (Exception e) {
      return failure(e);
    }
  }

  static <S, E, T, R> Function<T, Program<S, E, R>> lift(Function<T, R> mapper) {
    return mapper.andThen(Success::new);
  }

  static <S, E, T> Program<S, E, T> access(Function<S, T> mapper) {
    return new Access<>(mapper);
  }

  static <S, E, T, R> Program<S, E, R> pipe(Program<S, E, T> first, Function<T, Program<S, E, R>> next) {
    return first.flatMap(next);
  }

  static <S, E, F, T> Program<S, F, T> recover(Program<S, E, T> first, Function<E, Program<S, F, T>> next) {
    return first.flatMapError(next);
  }

  static <S, E, T, U, R> Program<S, E, R> pipe(
      Program<S, E, T> first, Function<T, Program<S, E, U>> second, Function<U, Program<S, E, R>> next) {
    return pipe(first, t -> pipe(second.apply(t), next));
  }

  static <S, E, T, U, V, R> Program<S, E, R> pipe(
      Program<S, E, T> first, Function<T, Program<S, E, U>> second, Function<U, Program<S, E, V>> third, Function<V, Program<S, E, R>> next) {
    return pipe(first, t -> pipe(second.apply(t), u -> pipe(third.apply(u), next)));
  }

  @SuppressWarnings("unchecked")
  default Either<E, T> eval(S state) {
    return switch (this) {
      case Success<S, E, T>(T value) -> Either.right(value);
      case Failure<S, E, T>(E error) -> Either.left(error);
      case FlatMap<S, E, ?, T>(var current, var next) -> {
        var result = current.eval(state);
        yield result.fold(
            _ -> (Either<E, T>) result,
            value -> {
              var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(value);
              return nextValue.eval(state);
            });
      }
      case FlatMapError<S, ?, E, T>(var current, var next) -> {
        var result = current.eval(state);
        yield result.fold(
            error -> {
              var nextValue = ((Function<Object, Program<S, E, T>>) next).apply(error);
              return nextValue.eval(state);
            },
            _ -> (Either<E, T>) result);
      }
      case Access<S, E, T>(var mapper) -> Either.right(mapper.apply(state));
    };
  }
}
```

```java
import static Program.access;
import static Program.pipe;

interface Console {

  interface Service {
    void println(String line);
    String readln();
  }

  static <S extends Console.Service, E> Program<S, E, String> readln() {
    return access(Console.Service::readln);
  }

  static <S extends Console.Service, E> Program<S, E, Void> println(String question) {
    return access(s -> {
      s.println(question);
      return null;
    });
  }

  static <S extends Console.Service, E> Program<S, E, String> prompt(String question) {
    return pipe(println(question), _ -> readln());
  }
}
```

```java
import static Program.access;

interface Random {

  interface Service {
    int nextInt(int bound);
  }

  static <S extends Random.Service, E> Program<S, E, Integer> nextInt(int bound) {
    return access(s -> s.nextInt(bound));
  }
}
```

```java
import static Program.access;

interface State {

  interface Service {
    void setValue(int value);
    int getValue();
  }

  static <S extends Service, E> Program<S, E, Void> setValue(int value) {
    return access(state -> {
      state.setValue(value);
      return null;
    });
  }

  static <S extends Service, E> Program<S, E, Integer> getValue() {
    return access(Service::getValue);
  }
}
```

```java
import static Console.println;
import static Console.prompt;
import static Program.attempt;
import static Program.lift;
import static Program.pipe;
import static Program.recover;
import static Random.nextInt;
import static State.getValue;

class Game {

  static void main() {
    game().eval(new Context());
  }

  static <S extends Console.Service & Random.Service & State.Service> Program<S, Exception, Void> game() {
    return pipe(
        prompt("Do you want to play a game? (y/n)"),
        answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return pipe(generateNumber(), _ -> play());
          }
          return println("Bye!");
        });
  }

  static <S extends Console.Service & State.Service> Program<S, Exception, Void> play() {
    return pipe(
        readNumber(),
        Game::checkNumber,
        result -> {
          if (result) {
            return println("YOU WIN!");
          }
          return play();
        });
  }

  static <S extends Console.Service> Program<S, Exception, Integer> readNumber() {
    return recover(
        pipe(
          prompt("Enter a number between 0 to 9"),
          s -> attempt(() -> Integer.parseInt(s))
        ),
      _ -> pipe(println("Invalid value"), _ -> readNumber()));
  }

  static <S extends Random.Service & State.Service> Program<S, Exception, Void> generateNumber() {
    return pipe(
        nextInt(10),
        State::setValue
      );
  }

  static <S extends State.Service, E> Program<S, Exception, Boolean> checkNumber(Integer number) {
    return pipe(
        getValue(),
        lift(value -> value == number)
      );
  }
}
```

```java
import java.util.concurrent.ThreadLocalRandom;

public class Context implements Console.Service, State.Service, Random.Service {

  private int value;

  @Override
  public void setValue(int value) {
    this.value = value;
  }

  @Override
  public int getValue() {
    return value;
  }

  @Override
  public int nextInt(int bound) {
    return ThreadLocalRandom.current().nextInt(bound);
  }

  @Override
  public void println(String line) {
    IO.println(line);
  }

  @Override
  public String readln() {
    return IO.readln();
  }
}
```

```java
import java.util.function.Function;

sealed interface Either<L, R> {

  record Left<L, R>(L left) implements Either<L, R> {}
  record Right<L, R>(R right) implements Either<L, R> {}

  default <T> T fold(Function<L, T> leftCase, Function<R, T> rightCase) {
    return switch (this) {
      case Left<L, R>(var left) -> leftCase.apply(left);
      case Right<L, R>(var right) -> rightCase.apply(right);
    };
  }

  static <L, R> Either<L, R> left(L left) {
    return new Left<>(left);
  }

  static <L, R> Either<L, R> right(R right) {
    return new Right<>(right);
  }
}
```

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static v3.Game.game;

import java.util.ArrayList;
import java.util.List;

import org.junit.jupiter.api.Test;

class GameTest {

  @Test
  void happyPath() {
    var context = new MockContext(2, "y", "1", "2");

    game().eval(context);

    var expectedOutput = List.of(
        "Do you want to play a game? (y/n)",
        "Enter a number between 0 to 9",
        "Enter a number between 0 to 9",
        "YOU WIN!");
    assertEquals(expectedOutput, context.output);
  }

  @Test
  void userDontWantToPlay() {
    var context = new MockContext(2, "n");

    game().eval(context);

    var expectedOutput = List.of(
        "Do you want to play a game? (y/n)",
        "Bye!");
    assertEquals(expectedOutput, context.output);
  }

  @Test
  void userEnterAnInvalidValue() {
    var context = new MockContext(3, "y", "c", "3");

    game().eval(context);

    var expectedOutput = List.of(
        "Do you want to play a game? (y/n)",
        "Enter a number between 0 to 9",
        "Invalid value",
        "Enter a number between 0 to 9",
        "YOU WIN!");
    assertEquals(expectedOutput, context.output);
  }

  class MockContext implements Console.Service, Random.Service, State.Service {

    private final List<String> output = new ArrayList<>();
    private final List<String> input = new ArrayList<>();

    private int value;
    private int number;

    public MockContext(int number, String... input) {
      this.number = number;
      this.input.addAll(List.of(input));
    }

    @Override
    public void setValue(int value) {
      this.value = value;
    }

    @Override
    public int getValue() {
      return value;
    }

    @Override
    public int nextInt(int bound) {
      return number;
    }

    @Override
    public void println(String line) {
      output.addLast(line);
    }

    @Override
    public String readln() {
      return input.removeFirst();
    }
  }
}
```