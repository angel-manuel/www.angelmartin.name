---
layout: post
title: "JavaScript es muy bueno"
post_ref: javascript_is_great
lang: es
categories:
tags:
  - javascript
  - programming languages
---

JavaScript es muy bueno.

{% highlight javascript %}
console.log("I'm pretty good!")
{% endhighlight %}

Hablo completamente en serio.

Criticar JavaScript es un lugar común entre desarrolladores. Este lenguaje
feo, hecho a base de remiendos, con un nombre que lleva confundiendo a gente de RR.HH decadas,
enfrentado a las leyes más elementales de la lógica aristotélica. Este lenguaje que solo
usamos porque es la única alternativa en el navegador web.
Pero olvidemos por un momento lo malo y pensemos en lo bueno.

## Programación Funcional

De entre los lenguajes de programación más usados JavaScript
es especial por tener funciones como valores desde el primer día, allá en 1995.
Muchos otros lenguajes han ido incorporando funcionalidad parecida poco a poco
(lambdas en Java 8, C++11, C# 3.0, PHP 5).

De hecho, la programación funcional, comunmente reconocida como limpia y elegante,
es un estilo completamente natural y promovido por el propio lenguaje.
El [paquete del que más software depende en npm](https://gist.github.com/anvaka/8e8fa57c7ee1350e3491#file-01-most-dependent-upon-md)
es [lodash](https://lodash.com/), una libreria llena de utilidades para la programación
funcional y con una [ascendencencia](https://underscorejs.org/) que se [remonta](http://prototypejs.org/) largo atras.

## Procesamiento asíncrono

"Funciones! ¿No te referiras a callbacks? Generan código horrible".

Todo el mundo esta de acuerdo, includo el comité que evoluciona JavaScript, razón por
la que han arreglado el problema introduciendo promesas y async/await.
Porque JavaScript también es único en ser asíncrono de nacimiento. Escribir código
asíncrono en cualquier otro lenguaje supone desviarse del camino idiomático natural,
en JavaScript el asincronismo es el camino. Esto es por lo que Node.js puede brillar
tanto en el lado del servidor(y sin necesidad de tunear mpm_prefork_module).

```javascript
// Boring and nested
function doThing(callback) {
    talkToBackend(function (resultA) {
        talkTo3rdParty(resultA, function (resultB) {
            callback(resultA + resultB)
        })
    })
}

// Clean and slick
async function doThing() {
    const resultA = await talkToBackend()
    const resultB = await talkTo3rdParty()

    return resultA + resultB
}

// Parallel yeah.
// Don't even bother writing this with callbacks.
async function doThingParallel() {
    const [resultA, resultB] = await Promise.all([
        talkToBackend,
        talkTo3rdParty,
    ])

    return resultA + resultB
}
```

## Sobre esas conversiones...

"Vale. Pero sigue teniendo [cosas estupidas](https://www.destroyallsoftware.com/talks/wat). Por ejemplo aquí rompe la transitividad de la igualdad, y allá rompe la conmutatividad de la suma"

Sí, algunas de esas cosas son un poco vergonzosas y otras son bastante graciosas.
Practicamente todas esas anomalías proceden de conversiones de tipos implicitas y bastante creativas
a string y a booleanos. Es una decisión de diseño entendible, la web no está fuertemente tipada,
todo lo que acaba en el DOM es una string, por lo tanto crear un lenguaje ergonómico para la web
conlleva cierta liberalidad a la hora de convertir strings en un sentido y otro.

Esto no significa que podamos evitar ciertos dolores, para empezar, usar la comparación estricta "==="
casi siempre es mejor idea que "==". Es fácil configurar nuestro linter para forzarnos a usar la mejor opción.
Pero podemos hacer aún mejor...

## JavaScript es bueno, TypeScript es genial

TypeScript es una extensión de JavaScript con tipado opcional

..., y es maravilloso. 
Con el mínimo esfuerzo resuelve casi todos los problemas debidos a conversiones desafortunadas, permite autocompletado profundo,
soporte de clases, modulos, enumeraciones para cualquier navegador. Creo que no hay ningun lenguaje en el
que programe más rápido y con menos errores que TypeScript.

## El ecosistema es inigualable

JavaScript tiene un ecosistema gigantesco, cualquiera lo tendría díficil para mencionar una función
para la que ya no exista una libreria open source, muchas veces de gran calidad y con mantenimiento activo.
Los frameworks modernos permiten una rápidez y facilidad sorprendentes. Hoy en día JS en el único stack tecnologico
que permite ir rapido y llegar a la web, al smartphone, al servidor y hasta al escritorio.

No significa que haya problemas, el cambiar de herramientas según las modas, la cantidad de dependencias de
cualquier proyecto con tamaño, la complejidad innesaria que a veces nos imponemos. Y aun así estos son problemas
derivados del exito, estoy seguro de que desarrolladores en nichos más oscuros les encantaría cambiar sus problemas
por estos.

## JavaScript to the moon

JavaScript ha sobrevivido a la promesa de Java "Write Once, Run Everywhere"(quizas el nombre no era tan malo).
Incluso así no para de moverse, asm.js, WebAssembly, ES.Next, WebGL, LocalStorage, ... Es increible que un lenguaje
ideado para complementar ligeramente la web se haya demostrado tan versatil, ya hay [a quien no le extraña que JavaScript
llegase a Ring 0](https://www.destroyallsoftware.com/talks/the-birth-and-death-of-javascript)(sí, el autor de [Wat](https://www.destroyallsoftware.com/talks/wat)).

## Conclusión

Todo esto no significa que JavaScript no tenga puntos oscuros y dificultades. No es díficil encontrarlos:
null vs undefined, monkey patching, "[Object object]", leaks de memoria y otros recursos, callbacks en callbacks,
multiples platformas, APIs viejas e inconsistentes, etc

Pero en balance tenemos suerte de que JavaScript sea el lenguaje de la web, despues de que Brendan Eich crease
la primera versión en 10 días.

Quizás JavaScript no nos ama locamente, pero ciertamente no nos odia, yo creo que hasta le caemos bien,
y a nostros debería caernos bien también.
