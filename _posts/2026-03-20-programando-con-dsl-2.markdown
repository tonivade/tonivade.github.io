---
layout: post
title:  Programando con DSLs en Java 25 (II)
date:   2026-03-19 21:00:00
categories: programming java functional-programming dsl monad
---

El otro día veíamos que para desarrollar un DSLs aparecía de manera natural una mónada 
(en concreto una free monad pero no quiero entrar en esto ahora).

Hoy vamos a trabajar con otro ejemplo de tal manera que tengamos no solo que manejar
los resultados de una operación y pasársela a la siguiente, sino también necesitaremos
manejar un estado.

Un juego muy sencillo, el ordenador pensará un número del 0 al 9 y nosotros tendremos
que adivinarlo. La dinámica será la siguiente:

1. Primero el programa nos preguntará si queremos jugar a un juego.
1. Si respondemos que si, pensará un número de 0 al 9 y nos preguntará qué numero es.
1. Si no acertamos volverá ha hacernos la misma pregunta.
1. Si acertamos, imprime por consola que hemos ganado y se termina.

```
=> Do you want to play a game? (y/n)
<= y
=> Enter a number between 0 to 9
<= 1
=> Enter a number between 0 to 9
<= 2
=> YOU WIN!
```

Para implementar un DSL para este juego necesitaremos definir varias operaciones:

1. Operaciones de lectura/escritura en consola, lo mismo que hicimos en el anterior artículo.
1. Necesitamos generar un número aleatorio de 0 a 9
1. Necesitamos guardar ese valor en alguna parte y luego recuperarlo para saber si el usuario
ha acertado.
1. Y por último necesitamos el pegamento para combinar las diferentes operaciones.

Estas serían las operaciones:

```java
sealed interface Game {
  record WriteLine(String line) implements Game {}
  record ReadLine() implements Game {}
  record NextInt(int bound) implements Game {}
  record GetValue() implements Game {}
  record SetValue(int value) implements Game {}
  record AndThen(Game current, Function<?, Game> next) implements Game {}
}
```

Todavía no está completo, necesitamos ir rellenando huecos. En `AndThen` la función `next` define `?`
porque no sabemos qué tipo de dato va llegar. Además, las diferentes operaciones devuelven
tipos diferentes, `ReadLine` devolverá un `String`, y `NextInt` un `Integer`. Necesitamos
parametrizar nuestro DSL.

```java
sealed interface Game<T> {
  record WriteLine(String line) implements Game<Void> {}
  record ReadLine() implements Game<String> {}
  record NextInt(int bound) implements Game<Integer> {}
  record GetValue() implements Game<Integer> {}
  record SetValue(int value) implements Game<Void> {}
  record AndThen<T>(Game current, Function<?, Game> next) implements Game<T> {}
}
```

Mucho mejor, ahora vemos claro qué devuelve cada operación. `WriteLine` y `SetValue`, que realizan
side effects, no devuelven nada, en java para modelar ese caso devolvemos `Void`. `ReadLine` devuelve
un `String` y `NextInt` y `GetValue` devolverán `Integer`. Pero seguimos sin saber que tipo de dato 
de entrada tendremos en `AndThen`, por ahora solo sabemos que devuelve un valor `T`.

```java
sealed interface Game<T> {
  //...
  record AndThen<X, T>(Game<X> current, Function<X, Game<T>> next) implements Game<T> {}
}
```

Por lo tanto, en caso de `AndThen` necesitamos definir otro parámetro para definir el tipo de entrada.
Lo hemos llamado `X`. Y por último la función devolverá un `Game<T>`.

Ahora empecemos a escribir el programa, aunque primero voy a definir un método de ayuda para 
hacer el código más legible.

```java
static Game<String> prompt(String question) {
  return new WriteLine(question).andThen(_ -> new ReadLine());
}
```

Simplemente este método combina un `WriteLine` y un `ReadLine`.

```java
static void main() {
  var program = 
    prompt("Do you want to play a game? (y/n)")
      .andThen(answer -> {
        if (answer.equalsIgnoreCase("y")) {
          return new NextInt(10).andThen(SetValue::new).andThen(_ -> play());        
        }
        return new WriteLine("Bye!");
      });
}
```

Para hacerlo más legible, tendremos otro método `play` donde irá la lógica de acertar el número:

```java
static Game<Void> play() {
  return prompt("Enter a number between 0 to 9")
    .andThen(number -> {
      return new GetValue()
        .andThen(value -> value == Integer.parseInt(number));  // something is missing here
    })
    .andThen(result -> {
      if (result) {
        return new WriteLine("YOU WIN!");
      }
      return play();      
    });
}
```

Pero aquí todavía falta algo, `andThen` necesita devolver una instancia de `Game`, pero
ahora mismo devolvemos un `boolean`. ¿Cómo podemos solucionar esto?

```java
sealed interface Game<T> {
  //...
  record Done<T>(T value) implements Game<T> {}
}
```

Hemos definido otra operación llamada `Done` que simplemente recubre un valor cualquiera. 
Ahora podemos resolver nuestro problema de antes de esta manera.

```java
static Game<Void> play() {
  return prompt("Enter a number between 0 to 9")
    .andThen(number -> {
      return new GetValue()
        .andThen(value -> new Done<>(value == Integer.parseInt(number)));
    })
    .andThen(result -> {
      if (result) {
        return new WriteLine("YOU WIN!");
      }
      return play();      
    });
}
```

Podemos ir un paso más allá y definir otro método de ayuda para hacerlo más legible.

```java
default <R> Game<R> map(Function<T, R> mapper) {
  return andThen(mapper.andThen(Done::new));
}
```

Este es un método `map` muy similar al que tenemos en `Stream` u `Optional`.

```java
static Game<Void> play() {
  return prompt("Enter a number between 0 to 9")
    .map(Integer::parseInt)
    .andThen(number -> new GetValue().map(value -> value == number))
    .andThen(result -> {
      if (result) {
        return new WriteLine("YOU WIN!");
      }
      return play();
    });
}
```

Ya tenemos nuestro programa definido. Un detalle más, cuando el usuario no acierta
el número el método `play` se vuelve a llamar a si mismo, por lo que usando estamos
usando recursividad para simular un bucle.

Ahora necesitamos implementar la función para evaluar el programa, pero iremos paso 
a paso para no perdernos.

Primero: ¿qué tendrá que devolver el método `eval`? Pues el resultado de evaluar la 
operación que en nuestro caso será `T`.

```java
default T eval() {    
  // TBD
}
```

Ahora podemos aplicar pattern matching y empezar a evaluar operaciones. Primero las de
lectura/escritura en consola, lo mismo que hacíamos en el anterior artículo:

```java
default T eval() {
  return switch (this) {
    case WriteLine(var line) -> {
      IO.println(line);
      yield null;
    }
    case ReadLine _ -> IO.readln();
  };
}
```

Ahora la operación para generar un número aleatorio, usamos `ThreadLocalRandom` que resulta
muy cómoda para este tipo de cosas:

```java
default T eval() {
  return switch (this) {
    // ...
    case NextInt(var bound) -> ThreadLocalRandom.current().nextInt(bound);    
  };
}
```

Y ahora viene algo nuevo, necesitamos implementar las operaciones para guardar y recuperar
un valor. Necesitaremos algún sitio donde poder guardar esa información. Lo más fácil sería
simplemente pasar un objeto `Context` como parámetro al método `eval` donde poder guardar ese
valor.

```java
default T eval(Context context) {
  return switch (this) {
    // ...
    case SetValue(var value) -> {
      context.set(value);
      yield null;
    }
    case GetValue _ -> context.get();    
  };
}
```

Y por último necesitamos evaluar las dos operaciones `Done` y `AndThen` que son las que nos
sirven para combinar el resto de operaciones.

```java
default T eval(Context context) {
  return switch (this) {
    // ...
    case Done<T>(var value) -> value;
    case AndThen<?, T>(var current, var next) -> {
      var value = current.eval(context);
      yield next.apply(value).eval(context);      
    }
  };
}
```

Para el caso de `Done` es muy sencillo de implementar. Simplemente devolver el valor que tiene
dentro, sin más.

El problema está con `AndThen` y es porque desconocemos el tipo de entrada y el compilador se
queja de que no puede inferir el tipo y falla, eso es por que tenemos una `?` en su lugar y no 
tenemos manera de hacer esto limpiamente sin castings:

```java
default T eval(Context context) {
  return switch (this) {
    // ...
    case Done<T>(var value) -> value;
    case AndThen<?, T>(var current, var next) -> {
      var value = (Object) current.eval(context);
      var nextValue = ((Function<Object, Game<T>>) next).apply(value);
      yield nextValue.eval(context);
    }
  };
}
```

Podemos dejarlo más limpio si creamos un método privado en `AndThen` ya que ahí si que conocemos
los tipos y no es necesario realizar ningún tipo de casting. Pero eso lo dejo como ejercicio para 
el lector 😄.

Y por último, nos queda un pequeño cambio para dejar al compilador de Java conforme, ya que insiste 
que en el método `eval` los tipos no son compatibles y hay que hacer un cast. Todavía no entiendo 
por qué se queja el compilador ya que si estamos en un `Game<T>` el resultado de `eval` tendrá 
que ser `T` y no importa lo que hagamos ahí dentro que seguro que devolverá un `T`.

Pongo aquí el código completo del método `eval`.

```java
default T eval(Context context) {
  return (T) switch (this) { // here we have to cast to T, it's safe
    case WriteLine(var line) -> {
      IO.println(line);
      yield null;
    }
    case ReadLine _ -> IO.readln();
    case NextInt(var bound) -> ThreadLocalRandom.current().nextInt(bound);
    case SetValue(var value) -> {
      context.set(value);
      yield null;
    }
    case GetValue _ -> context.get();
    case Done<T>(var value) -> value;
    case AndThen<?, T>(var current, var next) -> {
      var value = (Object) current.eval(context);
      var nextValue = ((Function<Object, Game<T>>) next).apply(value);
      yield nextValue.eval(context);
    }
  };
}
```

Ahora bien, ¿qué pinta tendría la clase Context? Pues algo como esto podría valernos por ahora:

```java
final class Context {

  private int value;

  void set(int value) {
    this.value = value; 
  }

  int get() {
    return value; 
  }
}
```

Solo comentar que esta clase es mutable, lo que en cierto modo va en contra de los principios de
la programación funcional. Pero en este caso, las operaciones que modifican esta clase están 
definidas en nuestro DSL por lo que el estado solo cambiará dentro del flujo del programa. Por
lo que podemos decir que nadie salvo nosotros modificará el contexto.

Y qué conclusiones podemos sacar aquí? Pues que podemos generalizar todo esto y sacar factor
común, `Done` y `AndThen` son comunes a cualquier otro DSL. Las operaciones de consola
se podrían extraer y poderse reutilizar en otros programas. Lo mismo para las otras operaciones.
En el próximo artículo veremos como podemos hacerlo.

Y volviendo a lo que comentaba el principio, lo que hemos construido aquí es una free monad. Esto
qué es? Si vamos a la definición oficial, nos dice que nos permite crear una mónada a partir de
cualquier functor, pero esto a mi no me dice nada. Para mi, después de ver estos ejemplos con
los que hemos estado trabajando, es que con una free monad podemos combinar operaciones de cualquier
tipo para formar programas. Lo que más me gusta de esto es que al definir el programa nada
se ejecuta y que lo que tenemos es realmente es una descripción del programa, sería algo parecido
como al código fuente. Para que funcione hay que evaluarlo, y podemos evaluarlo de maneras diferentes
lo que nos da mucha flexibilidad.

Dejo aquí el código completo del ejemplo de hoy:

```java
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.Function;

sealed interface Game<T> {

  record WriteLine(String line) implements Game<Void> {}
  record ReadLine() implements Game<String> {}

  record NextInt(int bound) implements Game<Integer> {}

  record GetValue() implements Game<Integer> {}
  record SetValue(int value) implements Game<Void> {}

  record AndThen<X, T>(Game<X> current, Function<X, Game<T>> next) implements Game<T> {}
  record Done<T>(T value) implements Game<T> {}
  
  default <R> Game<R> map(Function<T, R> mapper) {
    return andThen(mapper.andThen(Done::new));
  }

  default <R> Game<R> andThen(Function<T, Game<R>> next) {
    return new AndThen<>(this, next);
  }
  
  static Game<String> prompt(String question) {
    return new WriteLine(question).andThen(_ -> new ReadLine());
  }
  
  static void main() {
    var program = 
      prompt("Do you want to play a game? (y/n)")
        .andThen(answer -> {
          if (answer.equalsIgnoreCase("y")) {
            return new NextInt(10).andThen(SetValue::new).andThen(_ -> play());        
          }
          return new WriteLine("Bye!");
        });
    
    program.eval(new Context());
  }
  
  static Game<Void> play() {
    return prompt("Enter a number between 0 to 9")
      .map(Integer::parseInt)
      .andThen(number -> new GetValue().map(value -> value == number))
      .andThen(result -> {
        if (result) {
          return new WriteLine("YOU WIN!");
        }
        return play();
      });
  }
  
  default T eval(Context context) {
    return (T) switch (this) {
      case WriteLine(var line) -> {
        IO.println(line);
        yield null;
      }
      case ReadLine _ -> IO.readln();
      case NextInt(var bound) -> ThreadLocalRandom.current().nextInt(bound);
      case SetValue(var value) -> {
        context.set(value);
        yield null;
      }
      case GetValue _ -> context.get();
      case Done<T>(var value) -> value;
      case AndThen<?, T>(var current, var next) -> {
        var value = (Object) current.eval(context);
        var nextValue = ((Function<Object, Game<T>>) next).apply(value);
        yield nextValue.eval(context);
      }
    };
  }
  
  final class Context {

    private int value;

    void set(int value) { 
      this.value = value; 
    }

    int get() { 
      return value; 
    }
  }
}
```