---
title: "Awesome functional programming en JavaScript — Spanish version"
layout: post
date: 2017-08-31 23:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- functional programming
- javascript
- typescript
- ramda
- sanctuary
category: blog
author: Ricardo Pallas
projects: true
description: FP state of the art in JS
# jemoji: '<img class="emoji" title=":ramen:" alt=":ramen:" src="https://assets.github.com/images/icons/emoji/unicode/1f35c.png" height="20" width="20" align="absmiddle">'
---

JavaScript es un lenguaje de programación multi-paradigma, casi siempre
utilizado orientado a objetos, aunque debido a su gran popularidad, se podría
decir que es el lenguaje de programación funcional (**FP**) más utilizado.

**Disclaimer:*** Este artículo no pretende enseñar ni introducir en profundidad
la programación funcional, sino que es una guía de bibliotecas y recursos, para
poder utilizar la mayoría de herramientas y capacidades que da el estilo de
programación funcional en JavaScript.*

![](https://cdn-images-1.medium.com/max/1600/1*5Ma0Z2oOwOsyJFAwT43kMg.png)
<span class="figcaption_hack">Lambda — Representa el [lambda
calculus](https://en.wikipedia.org/wiki/Lambda_calculus), base de la
programación funcional.</span>

#### ¿Qué es la programación funcional?

Empecemos con una breve introducción de la programación funcional para ponernos
en contexto.

La programación funcional es un paradigma de programación que se basa en
funciones modeladas como funciones matemáticas. La esencia de la programación
funcional es que los programas son una combinación de expresiones. Las
expresiones pueden ser valores concretos, variables o funciones. Las funciones
se pueden definir de forma más específica: son expresiones a las cuales se les
aplica un argumento o entrada, y una vez aplicadas, se pueden **reducir** o
**evaluar**. En los lenguajes funcionales y lenguajes modernos, las funciones
son ciudadanos de primer clase: se pueden utilizar como valores o pueden ser
pasadas como argumentos, o entradas, a otras funciones.

Cabe destacar que, los lenguajes puramente funcionales, están todos basados en
el [lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus).

#### ¿Por qué JavaScript?

Ciertamente, no es el mejor lenguaje para hacer FP, siendo que la corriente
actual suele programar en JavaScript de forma imperativa. Además, no es un
lenguaje de programación funcional
[puro](https://www.fpcomplete.com/blog/2017/04/pure-functional-programming), y
es débilmente y dinámicamente tipado 😥.

Sin embargo, sus puntos a favor son:

* Es uno de los lenguajes más utilizados en la industria y seguramente trabajes
con él.
* Lo más probable, es que ya sepas programar en JavaScript. No tienes que aprender
un lenguaje nuevo.
* Con la ayuda de bibliotecas puedes utilizar muchas de las herramientas de la FP.

### Bibliotecas (que facilitan la programación funcional)

Voy a hablar, en este punto, de unas pocas bibliotecas, que he elegido y
utilizado para FP en JavaScript.

#### Sanctuary

La primera de ellas (y mi favorita) es [Sanctuary](https://sanctuary.js.org/),
cuyo lema es *“El refugio del JavaScript inseguro”, *refiriéndose a que ayuda a
eliminar muchos errores en tiempo de ejecución, sobre todos los provocados por
valores nulos.

Sanctuary está inspirado por los lenguajes de programación Haskell y PureScript.
Provee un conjunto de funciones similares a Ramda y lodash-fp, pero muchas de
ellas son seguras y trabajan con *data types* directamente, como por ejemplo el
tipo Maybe.

Provee dos *data types* básicos, [Maybe](https://sanctuary.js.org/#maybe-type) y
[Either](https://sanctuary.js.org/#either-type), que cumplen la especificación
de [Fantasy Land](https://github.com/fantasyland/fantasy-land/tree/v3.3.0), que
es la especificación de facto de ADTs en JavaScript.

Promueve un estilo de programación más seguro, sin los odiosos null checks, y
reduce los posibles errores en tiempo de ejecución.

Una gran ventaja es su **sistema de tipos** ad-hoc en tiempo de ejecución,
definidos en
[sanctuary-def](https://github.com/sanctuary-js/sanctuary-def/tree/v0.12.1). Con
este sistema podemos detectar los errores causados por tipos de forma inmediata,
lo cual nos evita las clásicas sorpresas de JavaScript…

Conviene leer la [entrevista](https://survivejs.com/blog/sanctuary-interview/)
que se le hizo a su creador, donde se explica el por qué de la biblioteca.

#### Fluture

[Fluture](https://github.com/fluture-js/Fluture), es una biblioteca para provee
de una mónada para ejecutar código asíncrono, parecido a una promesa. Como tal,
representa un valor *success *o *failure, *que resulta de una operación
asíncrona de* *I/O. La diferencia es que Fluture es una mónada (cumple con su
interfaz y leyes), por lo tanto se evalúa de forma perezosa y no lanza los
side-effects al crearla.

Puede haber confusión entre las similitudes de una Promesa con una mónada. Se
podría decir que el *.then *es un *bind, *que el* resolve *es un* pure, etc.
*Pero no hay que olvidarse de que, una promesa, no ofrece la interfaz
especificada por Fantasy Land, ni mucho menos, las leyes de las mónadas. Además
de que ejecutan la operación asíncrona (side effects) nada más crearlas. Se
puede ver más claro en este
[artículo](https://glebbahmutov.com/blog/difference-between-promise-and-task/).

#### Daggy

[Daggy](https://github.com/fantasyland/daggy), es una pequeña, pero muy útil,
biblioteca cuya finalidad es crear
[ADTs](https://en.wikipedia.org/wiki/Algebraic_data_type) (También llamados
[Union Types](https://guide.elm-lang.org/types/union_types.html) por la
comunidad JS/ELM). Permiten representar datos complejos de forma natural e
incluso emular [pattern
matching](https://gist.github.com/yang-wei/4f563fbf81ff843e8b1e).

En el siguiente [ejemplo](https://codesandbox.io/s/3x47m2x0x6), se puede ver
una** brillante manera** de usar las
[ADTs](https://en.wikipedia.org/wiki/Algebraic_data_type) creadas con **Daggy**
en componentes de **React**.

#### Otras

Cabe destacar también, la biblioteca [RamdaJS](http://ramdajs.com/), que utilicé
en su día junto a Sanctuary. Es una biblioteca de utilidades, es parecida a
Underscore o lodash, pero ahora, Sanctuary es mucho más madura que al principio
y ha incorporado la mayoría de funciones que provee Ramda, y por ello, dejé de
usarla. Las principal diferencia entre ambas son como manejan las entradas
invalidas y el sistema de tipos en runtime. Ramda es más insegura (sus funciones
pueden causar excepciones) porque los creadores no quieren utilizar data types
(ellos creen que les quitaría usuarios), y solo proveen de funciones típicas en
programación funcional, como map, sin aprovechar todo el potencial de este
paradigma de programación.

Y FolktaleJS, una de las pioneras. Su versión 1.o siempre ha sido muy respetada
y ahora están haciendo un gran trabajo en la 2.0, reescribiéndola por completo.
Es el mismo concepto que Sanctuary, funciones tipo map, curry, chain, etc y
tipos de datos como Maybe, Either, Validation… Se podría decir que Folktale es
más orientado a un estilo Java y Sanctuary más hacia Haskell. Además Folktale
incluye el tipo *Task* para tareas asíncronas, muy parecido a lo que ofrece la
biblioteca Fluture.

### Libros y artículos

* [Professor Frisby’s Mostly Adequate Guide to Functional
Programming](https://github.com/MostlyAdequate/mostly-adequate-guide) — Libro
por excelencia en FP en JS, escrito por [Brian
Lonsdorf](https://twitter.com/drboolean). Introduce al paradigma de programación
funcional **en general** utilizando JavaScript. Es una introducción práctica,
que va añadiendo, desde la intuición, ejemplos reales. Es un libro
imprescindible si no se tiene experiencia previa con FP.
* [Functional-Light JavaScript](https://github.com/getify/functional-light-js) —
Este libro explora aquellos principios básicos de la FP que se pueden aplicar en
JS. Se diferencia en su enfoque práctico, sin usar toda la terminología, que a
muchos les echa atrás.
* [Why Curry Helps](https://hughfdjackson.com/javascript/why-curry-helps/) — Una
visión general de como el currying ayuda a escribir código mas reusable y
declarativo.
* [Functional Mumbo Jumbo —
ADTs](http://blog.jenkster.com/2016/06/functional-mumbo-jumbo-adts.html) — Una
introducción a los tipos algebraicos de datos, para principiantes.

### Ejemplos

* En el [repositorio](https://github.com/RPallas92/congreso-web-2016) del taller
que di en el Congreso web en 2016 acerca de la programación funcional en JS,
podrás encontrar las
[slides](https://github.com/RPallas92/congreso-web-2016/blob/master/slides.pdf)
y el [proyecto de
ejemplo](https://github.com/RPallas92/congreso-web-2016/tree/master/app_videos/app_finished),
un **buscador de vídeos de Youtube (utilizando React y FP).**
* [Escape from Callback
Mountain](https://github.com/justsml/escape-from-callback-mountain) —
Refactorización, diseño y buenas prácticas.
* Design & refactoring tips for Promise-based Functional JavaScript. Key benefits
include better readability, testability, and reusability. MIT.

### **Otros recursos**

El repositorio de [Awesome FP JS](https://github.com/stoeffel/awesome-fp-js)
provee de muchos recursos además de los aquí comentados.

### Bola extra: TypeScript

Sólo comentar que se [Giulio Canti](https://github.com/gcanti) está en proceso
de creación de la biblioteca [FP-TS](https://github.com/gcanti/fp-ts), que
permitirá el uso de la **programación funcional en TypeScript**, con la gran
mejora sobre JavaScript, de los tipos, que ayudan a escribir código más
correcto. Gracias a estos, se puede emular cierto grado de **pattern matching en
TypeScript**, sin bibliotecas adicionales, como se explica
[aquí.](https://pattern-matching-with-typescript.alabor.me/)

### Conclusión

Con la combinación de las bibliotecas **Sanctuary, Fluture, Daggy**, podemos
llegar a programar en el paradigma de programación funcional de una manera muy
similar a la que lo haríamos en cualquier otro lenguaje funcional, salvando las
distancias.

Con este artículo quería demostrar que es posible poner en práctica los
conceptos y herramientas de la FP, sin tener que recurrir a un lenguaje
especializado.

Por otro lado quiero decir, que por sólo hacer FP no vas a desarrollar programas
perfectos, sino que te expones a los mismos problemas que si no haces FP… pero
que aprender FP te va a ayudar a aprender nuevos conceptos para ser mejor
programador… como se discute
[aquí](https://medium.com/@bfil/just-enough-functional-programming-a0c4fd09c8f7).

También quiero recomendar el uso en desarrollos frontend de otros lenguajes
tipados como [PureScript](http://www.purescript.org/), Elm, ReasonML, que ayudan
a escribir programas más correctos, sobre todo cuando se trata de proyectos más
grandes y complejos.

Antes de finalizar, decir que en los enlaces que se han ido poniendo a lo largo
del artículo de puede ampliar más información.

Finalmente, para el siguiente post, veremos un ejemplo sencillo de como parsear
JSON de un servidor de forma segura con programación imperativa vs programación
funcional.