---
layout: post
title:  Programando con DSLs en Java 25 (II)
date:   2026-03-19 21:00:00
categories: programming java functional-programming dsl monad
---

El otro día veíamos que para desarrollar un DSLs aparecía de manera natural una mónada 
(en concreto una free monad pero no quiero entrar en eso). Hoy veremos cómo generalizar
este código y hacer algo que sea reusable.

Hoy vamos a trabajar con otro ejemplo de tal manera que tengamos que no solo manejar
los resultados de una operación y pasársela a la siguiente, sino también necesitaremos
manejar un estado.

Un juego muy sencillo, el ordenador pensará un número del 0 al 9 y nosotros tendremos
que adivinarlo. La dinámica sería la siguiente:

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

Esto no está completo, necesitamos ir rellenando huecos. La función next define `?` porque sabemos qué tipo de dato va llegar. Además, las diferentes operaciones devuelve
tipos diferentes, `ReadLine` devolverá un `String`, y NextInt un `Integer`. Necesitamos
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

Mucho mejor, pero seguimos sin saber que tipo de dato de entrada tendremos en `AndThen`,
por ahora solo sabemos que devuelve un valor `T`.

```java
sealed interface Game<T> {
  //...
  record AndThen<X, T>(Game<X> current, Function<X, Game> next) implements Game<T> {}
}
```

Por lo tanto, necesitamos definir otro parámetro para definir el tipo de entrada.

Ahora empecemos a escribir el programa, aunque primero voy a definir un método de ayuda para hacer el código más legible.

```java
static Game<String> prompt(String question) {
  return new WriteLine(question).andThen(_ -> new ReadLine());
}
```

Simplemente este método combina un `WriteLine` y un `ReadLine`.

```java
static void main() {
  prompt("Do you want to play a game? (y/n)")
    .andThen(answer -> {
      if (answer.equalsIgnoreCase("y")) {
        return new NextInt(10).andThen(SetValue::new).andThen(_ -> play());        
      }
      
      return new WriteLine("Bye!");
    });
}
```

Para hacerlo más legible habrá otro método `play` donde va la lógica de acertar el número:

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

Pero aquí todavía falta algo, `andThen` necesita devolver una instancia de Game, pero
ahora mismo devolvemos un `boolean`. ¿Cómo podemos solucionar esto?

```java
sealed interface Game<T> {
  //...
  record Done<T>(T value) implements Game<T> {}
}
```

Hemos definido otra operación llamada `Done` que simplemente recubre un valor cualquiera. Ahora podemos resolver nuestro problema de antes de esta manera.

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

Ya tenemos nuestro programa definido, ahora necesitamos implementar la función para 
evaluar el programa, pero iremos paso a paso para no perdernos.

Primero qué tendrá que devolver el método `eval`, pues el resultado de evaluar la 
operación que en nuestro caso será `T`.

```java
sealed interface Game<T> {
  default T eval() {    
    // ???
  }
}
```

Ahora podemos aplicar pattern matching y empezar a evaluar operaciones. Primero las de
lectura/escritura en consola, lo mismo que teníamos antes:

```java
sealed interface Game<T> {
  default T eval() {
    return switch (this) {
      case WriteLine(var line) -> {
        IO.println(line);
        yield null;
      }
      case ReadLine _ -> IO.readln();
    };
  }
}
```

Ahora la operación para generar un número aleatorio:

```java
sealed interface Game<T> {
  default T eval() {
    return switch (this) {
      // ...
      case NextInt(var bound) -> ThreadLocalRandom.current().nextInt(bound);    
    };
  }
}
```

Y ahora viene algo nuevo, necesitamos implementar las operaciones para guardar y recuperar
un valor. Necesitaremos algún sitio donde poder guardar esa información. Lo más fácil sería
simplemente pasar un parámetro al método `eval` donde poder guardar ese contexto.

```java
sealed interface Game<T> {
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
}
```

Y por último necesitamos evaluar las dos operaciones `Done` y `AndThen` que son las que nos
sirven para combinar el resto de operaciones.

```java
sealed interface Game<T> {
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
}
```

Para el caso de `Done` es muy sencillo de implementar. Simplemente devolver el valor que tiene
dentro y listo.

El problema está con `AndThen` y es porque desconocemos el tipo de entrada, por lo que tenemos
una `?` en su lugar y no tenemos manera de hacer esto limpiamente sin castings:

```java
sealed interface Game<T> {
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
}
```

Podemos dejarlo más limpio si creamos un método privado en `AndThen` ya que ahí si que conocemos
los tipos y no es necesario realizar ningún casting. Pero eso lo dejo como ejercicio para el
lector.

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
la programación funcional, pero en este caso, las operaciones que modifican esta clase están 
definidas en nuestro DSL por lo que el estado solo cambiará dentro del flujo del programa.

Y por último nos queda un pequeño cambio para dejar al compilador de Java conforme ya que el método
`eval` insiste en que los tipos no son compatibles y hay que hacer un cast. Todavía no entiendo por qué se queja el compilador ya que si estamos en un `Game<T>` el resultado de `eval` tendrá que ser `T` y no importa lo que hagamos ahí dentro que seguro que devolverá un `T`.

Pongo aquí el código completo del método `eval`.

```java
default T eval(Context context) {
  return (T) switch (this) { // here we have to cast to T
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