# Functional-Light JavaScript
# Chapter 7: Closure vs. Object

Anos atrás, Anton van Straaten elaborou o que se tornou famoso e frequentemente citado [koan](https://www.merriam-webster.com/dictionary/koan) para ilustrar e provocar uma importante tensão entre closure e objetos:

> O venerável mestre Qc Na estava caminhando com seu estudante, Anton. Esperando
colocar seu mestre em uma discussão, Anton disse "Mestre, eu ouvi que
objetos são uma coisa muito boa - isso é verdade?" Qc Na olhou com pena
para seu estudante e respondeu, "Tolo pupilo - objetos são meramente cluseres
do 'Paraguai'".
>
> Cheteado, Anton despediu-se de seu mestre e retornou à sua célula,
motivo a estudar closures. Ele cuidadosamente leu toda: "Lambda: The
Ultimate..." séries de artigos e seus cogêneros, e implementou um pequeno
interpretador Scheme com um sistema de objetos baseado em clouseres.
Ele aprendeu muito, e foi informar seu mestre sobre seu progresso.
>
> Em sua nova caminhada com Qc Na, Anton tentou impressionar seu mestre
dizendo "Mestre, eu me dediquei em estudar a matéria, e agora eu entendo
que objetos são realmente closures do 'Paraguai'." Qc Na respondeu batendo em
Anton com sua vareta, dizendo "Quando você aprenderá? Closures são objetos do
'Paraguai'." Naquele momento, Anton ficou iluminado.
>
> -- Anton van Straaten 6/4/2003
>
> http://people.csail.mit.edu/gregs/ll1-discuss-archive-html/msg03277.html

A postagem original, while brief, has more context to the origin and motivations, and I strongly suggest you read that post to properly set your mindset for approaching this chapter.

Many people I've observed read this koan smirk at its clever wit but then move on without it changing much about their thinking. However, the purpose of a koan (from the Zen Buddhist perspective) is to prod the reader into wrestling with the contradictory truths therein. So, go back and read it again. Now read it again.

Which is it? Is a closure a poor man's object, or is an object a poor man's closure? Or neither? Or both? Is merely the take-away that closures and objects are in some way equivalent?

And what does any of this have to do with functional programming? Pull up a chair and ponder for a while. This chapter will be an interesting detour, an excursion if you will.

## The Same Page

First, let's make sure we're all on the same page when we refer to closures and objects. We're obviously in the context of how JavaScript deals with these two mechanisms, and specifically talking about simple function closure (see [Chapter 2, "Keeping Scope"](ch2.md/#keeping-scope)) and simple objects (collections of key-value pairs).

For the record, here's an illustration of a simple function closure:

```js
function outer() {
    var one = 1;
    var two = 2;

    return function inner(){
        return one + two;
    };
}

var three = outer();

three();            // 3
```

And an illustration of a simple object:

```js
var obj = {
    one: 1,
    two: 2
};

function three(outer) {
    return outer.one + outer.two;
}

three( obj );       // 3
```

Many people conjure lots of extra things when you mention "closure", such as the asynchronous callbacks or even the module pattern with encapsulation and information hiding. Similarly, "object" brings to mind classes, `this`, prototypes, and a whole slew of other utilities and patterns.

As we go along, we'll carefully address the parts of this external context that matter, but for now, try to just stick to the simplest interpretations of "closure" and "object" as illustrated here; it'll make our exploration less confusing.

## Look Alike

It may not be obvious how closures and objects are related. So let's explore their similarities first.

To frame this discussion, let me just briefly assert two things:

1. A programming language without closures can simulate them with objects instead.
2. A programming language without objects can simulate them with closures instead.

In other words, we can think of closures and objects as two different representations of a thing.

### State

Consider this code from before:

```js
function outer() {
    var one = 1;
    var two = 2;

    return function inner(){
        return one + two;
    };
}

var obj = {
    one: 1,
    two: 2
};
```

Both the scope closed over by `inner()` and the object `obj` contain two elements of state: `one` with value `1` and `two` with value `2`. Syntactically and mechanically, these representations of state are different. But conceptually, they're actually quite similar.

As a matter of fact, it's fairly straightforward to represent an object as a closure, or a closure as an object. Go ahead, try it yourself:

```js
var point = {
    x: 10,
    y: 12,
    z: 14
};
```

Did you come up with something like?

```js
function outer() {
    var x = 10;
    var y = 12;
    var z = 14;

    return function inner(){
        return [x,y,z];
    }
};

var point = outer();
```

**Note:** The `inner()` function creates and returns a new array (aka, an object!) each time it's called. That's because JS doesn't afford us any capability to `return` multiple values without encapsulating them in an object. That's not technically a violation of our object-as-closure task, because it's just an implementation detail of exposing/transporting values; the state tracking itself is still object-free. With ES6+ array destructuring, we can declaratively ignore this temporary intermediate array on the other side: `var [x,y,z] = point()`. From a developer ergonomics perspective, the values are stored individually and tracked via closure instead of objects.

What if we have nested objects?

```js
var person = {
    name: "Kyle Simpson",
    address: {
        street: "123 Easy St",
        city: "JS'ville",
        state: "ES"
    }
};
```

We could represent that same kind of state with nested closures:

```js
function outer() {
    var name = "Kyle Simpson";
    return middle();

    // ********************

    function middle() {
        var street = "123 Easy St";
        var city = "JS'ville";
        var state = "ES";

        return function inner(){
            return [name,street,city,state];
        };
    }
}

var person = outer();
```

Let's practice going the other direction, from closure to object:

```js
function point(x1,y1) {
    return function distFromPoint(x2,y2){
        return Math.sqrt(
            Math.pow( x2 - x1, 2 ) +
            Math.pow( y2 - y1, 2 )
        );
    };
}

var pointDistance = point( 1, 1 );

pointDistance( 4, 5 );      // 5
```

`distFromPoint(..)` is closed over `x1` and `y1`, but we could instead explicitly pass those values as an object:

```js
function pointDistance(point,x2,y2) {
    return Math.sqrt(
        Math.pow( x2 - point.x1, 2 ) +
        Math.pow( y2 - point.y1, 2 )
    );
};

pointDistance(
    { x1: 1, y1: 1 },
    4,  // x2
    5   // y2
);
// 5
```

The `point` object state explicitly passed in replaces the closure that implicitly held that state.

### Behavior, Too!

It's not just that objects and closures represent ways to express collections of state, but also that they can include behavior via functions/methods. Bundling data with its behavior has a fancy name: encapsulation.

Consider:

```js
function person(name,age) {
    return happyBirthday(){
        age++;
        console.log(
            `Happy ${age}th Birthday, ${name}!`
        );
    }
}

var birthdayBoy = person( "Kyle", 36 );

birthdayBoy();          // Happy 37th Birthday, Kyle!
```

The inner function `happyBirthday()` has closure over `name` and `age` so that the functionality therein is kept with the state.

We can achieve that same capability with a `this` binding to an object:

```js
var birthdayBoy = {
    name: "Kyle",
    age: 36,
    happyBirthday() {
        this.age++;
        console.log(
            `Happy ${this.age}th Birthday, ${this.name}!`
        );
    }
};

birthdayBoy.happyBirthday();
// Happy 37th Birthday, Kyle!
```

We're still expressing the encapsulation of state data with the `happyBirthday()` function, but with an object instead of a closure. And we don't have to explicitly pass in an object to a function (as with earlier examples); JavaScript's `this` binding easily creates an implicit binding.

Another way to analyze this relationship: a closure associates a single function with a set of state, whereas an object holding the same state can have any number of functions to operate on that state.

As a matter of fact, you could even expose multiple methods with a single closure as the interface. Consider a traditional object with two methods:

```js
var person = {
    firstName: "Kyle",
    lastName: "Simpson",
    first() {
        return this.firstName;
    },
    last() {
        return this.lastName;
    }
}

person.first() + " " + person.last();
// Kyle Simpson
```

Just using closure without objects, we could represent this program as:

```js
function createPerson(firstName,lastName) {
    return API;

    // ********************

    function API(methodName) {
        switch (methodName) {
            case "first":
                return first();
                break;
            case "last":
                return last();
                break;
        };
    }

    function first() {
        return firstName;
    }

    function last() {
        return lastName;
    }
}

var person = createPerson( "Kyle", "Simpson" );

person( "first" ) + " " + person( "last" );
// Kyle Simpson
```

While these programs look and feel a bit different ergonomically, they're actually just different implementation variations of the same program behavior.

### (Im)mutability

Many people will initially think that closures and objects behave differently with respect to mutability; closures protect from external mutation while objects do not. But, it turns out, both forms have identical mutation behavior.

That's because what we care about, as discussed in [Chapter 6](ch6.md), is **value** mutability, and this is a characteristic of the value itself, regardless of where or how it's assigned:

```js
function outer() {
    var x = 1;
    var y = [2,3];

    return function inner(){
        return [ x, y[0], y[1] ];
    };
}

var xyPublic = {
    x: 1,
    y: [2,3]
};
```

The value stored in the `x` lexical variable inside `outer()` is immutable -- remember, primitives like `2` are by definition immutable. But the value referenced by `y`, an array, is definitely mutable. The exact same goes for the `x` and `y` properties on `xyPublic`.

We can reinforce the point that objects and closures have no bearing on mutability by pointing out that `y` is itself an array, and thus we need to break this example down further:

```js
function outer() {
    var x = 1;
    return middle();

    // ********************

    function middle() {
        var y0 = 2;
        var y1 = 3;

        return function inner(){
            return [ x, y0, y1 ];
        };
    }
}

var xyPublic = {
    x: 1,
    y: {
        0: 2,
        1: 3
    }
};
```

If you think about it as "turtles (aka, objects) all the way down", at the lowest level, all state data is primitives, and all primitives are value-immutable.

Whether you represent this state with nested objects, or with nested closures, the values being held are all immutable.

### Isomorphic

The term "isomorphic" gets thrown around a lot in JavaScript these days, and it's usually used to refer to code that can be used/shared in both the server and the browser. I wrote a blog post a while back that calls bogus on that usage of this word "isomorphic", which actually has an explicit and important meaning that's being clouded.

Here's some selections from a part of that post:

> What does isomorphic mean? Well, we could talk about it in mathematical terms, or sociology, or biology. The general notion of isomorphism is that you have two things which are similar in structure but not the same.
>
> In all those usages, isomorphism is differentiated from equality in this way: two values are equal if they’re exactly identical in all ways, but they are isomorphic if they are represented differently but still have a 1-to-1, bi-directional mapping relationship.
>
> In other words, two things A and B would be isomorphic if you could map (convert) from A to B and then go back to A with the inverse mapping.

Recall in [Chapter 2, "Brief Math Review"](ch2.md/#brief-math-review), we discussed the mathematical definition of a function as being a mapping between inputs and outputs. We pointed out this is technically called a morphism. An isomorphism is a special case of bijective (aka, 2-way) morphism that requires not only that the mapping must be able to go in either direction, but also that the behavior is identical in either form.

But instead of thinking about numbers, let's relate isomorphism to code. Again quoting my blog post:

> [O] que seria JS isomórfico se houvesse algo assim? Bem, pode ser que você tenha um conjunto de código JS que é convertido em outro conjunto de código JS, e que (importante) você poderia converter do último de volta para o primeiro se quisesse.

Como afirmamos anteriormente com nossos exemplos de closures-como-objetos e objetos-como-closures, essas alternâncias representativas vão para ambos os lados. Nesse aspecto, elas são isomorfismos entre si.

Simplificando, closures e objetos são representações isomórficas de estado (e sua funcionalidade associada).

Da próxima vez que você ouvir alguém dizer "X é isomórfico a Y", o que eles querem dizer é "X e Y podem ser convertidos de um para o outro em qualquer direção, e não perder informações".

### Debaixo do capô

Então, podemos pensar em objetos como uma representação isomórfica de closure da perspectiva do código que poderíamos escrever. Mas também podemos observar que um sistema de closure poderia realmente ser implementado -- e provavelmente é -- com objetos!

Pense desta forma: no código a seguir, como o JS está controlando a variável `x` para `inner()` para continuar referenciando, bem depois que `outer()` já foi executado?

```js
function outer() {
    var x = 1;

    return function inner(){
        return x;
    };
}
```

Podemos imaginar que o escopo -- o conjunto de todas variáveis definidas -- de `outer()` é implementado como um objeto com propriedades. Então, conceitualmente, em algum lugar na memória, tem algo do tipo:

```js
scopeOfOuter = {
    x: 1
};
```

    E então para a função `inner()`, quando criada, recebe um objeto escopo (empty) chamado `scopeOfInner`, que está vinculado por meio de seu `[[Prototype]]` ao objeto `scopeOfOuter`, mais ou menos assim:

```js
scopeOfInner = {};
Object.setPrototypeOf( scopeOfInner, scopeOfOuter );
```

Então, dentro de `inner()`, quando faz referência à variável lexical `x`, na verdade é mais como:

```js
return scopeOfInner.x;
```

`scopeOfInner` não tem uma propriedade `x`, mas é `[[Prototype]]`-linkado a `scopeOfOuter`, que tem uma propriedade `x`. Acessar `scopeOfOuter.x` via delegação de protótipo resulta no valor `1` sendo retornado.

Dessa forma, podemos ver por que o escopo de `outer()` é preservado (via closure) mesmo depois de terminar: porque o objeto `scopeOfInner` é vinculado ao objeto `scopeOfOuter`, mantendo assim esse objeto e suas propriedades vivos e bem.

Agora, tudo isso é conceitual. Não estou dizendo literalmente que o mecanismo JS usa objetos e protótipos. Mas é totalmente plausível que ele *poderia* funcionar de forma semelhante.

Muitas linguagens de fato implementam closures via objetos. E outras linguagens implementam objetos em termos de closures. Mas deixaremos o leitor usar sua imaginação sobre como isso funcionaria.

## Duas estradas divergiam em uma floresta...

Então closures e objetos são equivalentes, certo? Não exatamente. Aposto que eles são mais parecidos do que você pensava antes de começar este capítulo, mas eles ainda têm diferenças importantes.

Essas diferenças não devem ser vistas como fraquezas ou argumentos contra o uso; essa é a perspectiva errada. Elas devem ser vistas como recursos e vantagens que tornam um ou outro mais adequado (e legível!) para uma determinada tarefa.

### Mutabilidade Estrutural

Conceitualmente, a estrutura de um closure não é mutável.

Em outras palavras, você nunca pode adicionar ou remover o estado de um closure. O closure é uma característica de onde as variáveis ​​são declaradas (fixadas no autor/tempo de compilação) e não é sensível a nenhuma condição de tempo de execução -- assumindo que você use o modo estrito e/ou evite usar truques como `eval(..)`, é claro!

**Observação:** O mecanismo JS poderia tecnicamente eliminar um closure para eliminar quaisquer variáveis ​​em seu escopo que não serão mais usadas, mas esta é uma otimização avançada que é transparente para o desenvolvedor. Se o mecanismo realmente faz esses tipos de otimizações, acho mais seguro para o desenvolvedor assumir que o closure é por escopo em vez de por variável. Se você não quer que ele permaneça, não feche sobre ele!

No entanto, os objetos por padrão são bastante mutáveis. Você pode adicionar ou remover (`delete`) livremente propriedades/índices de um objeto, desde que esse objeto não tenha sido congelado (`Object.freeze(..)`).

Pode ser uma vantagem do código poder rastrear mais (ou menos!) estado dependendo das condições de tempo de execução no programa.

Por exemplo, vamos imaginar rastrear os eventos de pressionamento de tecla em um jogo. Quase certamente, você pensará em usar um array para fazer isso:

```js
function trackEvent(evt,keypresses = []) {
    return [ ...keypresses, evt ];
}

var keypresses = trackEvent( newEvent1 );

keypresses = trackEvent( newEvent2, keypresses );
```

**Nota:** Você percebeu por que eu não dei `push(..)` diretamente para `keypresses`? Porque em FP, normalmente queremos tratar arrays como estruturas de dados imutáveis ​​que podem ser recriadas e adicionadas, mas não alteradas diretamente. Trocamos o mal dos efeitos colaterais por uma reatribuição explícita (mais sobre isso depois).

Embora não estejamos mudando a estrutura do array, poderíamos se quiséssemos. Mais sobre isso em um momento.

Mas um array não é a única maneira de rastrear essa "lista" crescente de objetos `evt`. Poderíamos usar o closure:

```js
function trackEvent(evt,keypresses = () => []) {
    return function newKeypresses() {
        return [ ...keypresses(), evt ];
    };
}

var keypresses = trackEvent( newEvent1 );

keypresses = trackEvent( newEvent2, keypresses );
```

Você percebe o que está acontecendo aqui?

Cada vez que adicionamos um novo evento à "lista", criamos um novo closure envolto em torno da função `keypresses()` existente (closure), que captura o `evt` atual. Quando chamamos a função `keypresses()`, ela chamará sucessivamente todas as funções aninhadas, construindo uma matriz intermediária de todos os objetos `evt` fechados individualmente. Novamente, o closure é o mecanismo que rastreia todo o estado; a matriz que você vê é apenas um detalhe de implementação da necessidade de uma maneira de retornar vários valores de uma função.

Então, qual é mais adequado para nossa tarefa? Nenhuma surpresa aqui, a abordagem de matriz é provavelmente muito mais apropriada. A imutabilidade estrutural de um closure significa que nossa única opção é envolver mais closures em torno dele. Os objetos são extensíveis por padrão, então podemos apenas aumentar a matriz conforme necessário.

A propósito, embora eu esteja apresentando essa (i)mutabilidade estrutural como uma diferença clara entre closure e object, a maneira como estamos usando o object como um valor imutável é, na verdade, mais similar do que não.

Criar um novo array para cada adição ao array é tratar o array como estruturalmente imutável, o que é conceitualmente simétrico ao closure ser estruturalmente imutável por seu próprio design.

### Privacidade

Provavelmente uma das primeiras diferenças que você pensa ao analisar closure vs. object é que closure oferece "privacidade" de estado por meio de escopo lexical aninhado, enquanto objects expõem tudo como propriedades públicas. Essa privacidade tem um nome chique: ocultação de informações.

Considere a ocultação de closure lexical:

```js
function outer() {
    var x = 1;

    return function inner(){
        return x;
    };
}

var xHidden = outer();

xHidden();          // 1
```

Now the same state in public:

```js
var xPublic = {
    x: 1
};

xPublic.x;          // 1
```

Existem algumas diferenças óbvias em torno dos princípios gerais da engenharia de software — considere a abstração, o padrão de módulo com APIs públicas e privadas, etc. — mas vamos tentar restringir nossa discussão à perspectiva da FP; afinal, este é um livro sobre programação funcional!

#### Visibilidade

Pode parecer que a capacidade de ocultar informações é uma característica desejada do rastreamento de estado, mas acredito que o FPer pode argumentar o oposto.

Uma das vantagens de gerenciar o estado como propriedades públicas em um objeto é que é mais fácil enumerar (e iterar!) todos os dados em seu estado. Imagine que você quisesse processar cada evento de pressionamento de tecla (do exemplo anterior) para salvá-lo em um banco de dados, usando um utilitário como:

```js
function recordKeypress(keypressEvt) {
    // database utility
    DB.store( "keypress-events", keypressEvt );
}
```

Se você já tem um array — apenas um objeto com propriedades públicas nomeadas numericamente — isso é muito simples usando um utilitário de array JS integrado `forEach(..)`:

```js
keypresses.forEach( recordKeypress );
```

Mas se a lista de pressionamentos de tecla estiver oculta dentro do closure, você terá que expor um utilitário na API pública do closure com acesso privilegiado aos dados ocultos.

Por exemplo, podemos dar ao nosso exemplo closure-`keypresses` seu próprio `forEach`, como arrays internos têm:

```js
function trackEvent(
    evt,
    keypresses = {
        list() { return []; },
        forEach() {}
    }
) {
    return {
        list() {
            return [ ...keypresses.list(), evt ];
        },
        forEach(fn) {
            keypresses.forEach( fn );
            fn( evt );
        }
    };
}

// ..

keypresses.list();      // [ evt, evt, .. ]

keypresses.forEach( recordKeypress );
```

A visibilidade dos dados de estado de um objeto torna seu uso mais direto, enquanto o closure obscurece o estado, fazendo com que trabalhemos mais para processá-lo.

#### Controle de mudança

Se a variável lexical `x` estiver oculta dentro de um closure, o único código que tem a liberdade de reatribuí-la também está dentro desse closure; é impossível modificar `x` de fora.

Como vimos no [Capítulo 6](ch6.md), esse fato por si só melhora a legibilidade do código ao reduzir a área de superfície que o leitor deve considerar para prever o comportamento de qualquer variável dada.

A proximidade local da reatribuição lexical é um grande motivo pelo qual não acho `const` como um recurso tão útil. Os escopos (e, portanto, os closures) devem, em geral, ser bem pequenos, e isso significa que haverá apenas algumas linhas de código que podem afetar a reatribuição. Em `outer()` acima, podemos inspecionar rapidamente para ver se nenhuma linha de código reatribui `x`, então, para todos os efeitos, ele está agindo como uma constante.

Esse tipo de garantia é um poderoso contribuidor para nossa confiança na pureza de uma função, por exemplo.

Por outro lado, `xPublic.x` é uma propriedade pública, e qualquer parte do programa que obtém uma referência a `xPublic` tem a capacidade, por padrão, de reatribuir `xPublic.x` a algum outro valor. Isso é muito mais linhas de código para considerar!

É por isso que no [Capítulo 6, vimos `Object.freeze(..)`](ch6.md/#its-freezing-in-here) como um meio rápido e prático de tornar todas as propriedades de um objeto para somente leitura (`writable: false`), para que elas não possam ser reatribuídas de forma imprevisível.

Infelizmente, `Object.freeze(..)` é tudo ou nada e irreversível.

Com o closure, você tem algum código com o privilégio de alterar, e o resto do programa é restrito. Quando você congela um objeto, nenhuma parte do código poderá ser reatribuída. Além disso, uma vez que um objeto é congelado, ele não pode ser descongelado, então as propriedades permanecerão somente leitura durante a duração do programa.

Em lugares onde eu quero permitir a reatribuição, mas restringir sua área de superfície, os closures são uma forma mais conveniente e flexível do que objetos. Em lugares onde eu não quero nenhuma reatribuição, um objeto congelado é muito mais conveniente do que repetir declarações `const` em toda a minha função.

Muitos FPers assumem uma postura linha-dura sobre reatribuição: ela não deve ser usada. Eles tendem a usar `const` para tornar todas as variáveis ​​de closure para somente leitura, e eles usarão `Object.freeze(..)` ou estruturas de dados totalmente imutáveis ​​para evitar a reatribuição de propriedades. Além disso, eles tentarão reduzir a quantidade de variáveis ​​e propriedades explicitamente declaradas/rastreadas sempre que possível, preferindo transferência de valor — cadeias de funções, valor `return` passado como argumento, etc. — em vez de armazenamento de valor intermediário.

Este livro é sobre programação "Functional-Light" em JavaScript, e este é um daqueles casos em que eu me desvio da multidão do FP principal.

Eu acho que a reatribuição de variáveis ​​pode ser bastante útil e, quando usada apropriadamente, bastante legível em sua explicitude. Certamente tem sido minha experiência que a depuração é muito mais fácil quando você pode inserir um `debugger` ou breakpoint, ou rastrear uma expressão watch.

### Estado de clonagem

Como aprendemos no [Capítulo 6](ch6.md), uma das melhores maneiras de evitar que efeitos colaterais corroam a previsibilidade do nosso código é garantir que tratamos todos os valores de estado como imutáveis, independentemente de serem realmente imutáveis ​​(congelados) ou não.

Se você não estiver usando uma biblioteca criada especificamente para fornecer estruturas de dados imutáveis ​​sofisticadas, a abordagem mais simples será suficiente: duplique seus objetos/matrizes sempre antes de fazer uma alteração.

Matrizes são fáceis de clonar superficialmente -- basta usar `...` array spread:

```js
var a = [ 1, 2, 3 ];

var b = [...a];
b.push( 4 );

a;          // [1,2,3]
b;          // [1,2,3,4]
```

Objetos também podem ser clonados relativamente facilmente:

```js
var o = {
    x: 1,
    y: 2
};

// in ES2018+, using object spread:
var p = { ...o };
p.y = 3;

// in ES6/ES2015+:
var p = Object.assign( {}, o );
p.y = 3;
```

Se os valores em um objeto/matriz forem eles próprios não primitivos (objetos/matriz), para obter uma clonagem profunda, você terá que percorrer cada camada manualmente para clonar cada objeto aninhado. Caso contrário, você terá cópias de referências compartilhadas para esses subobjetos, e isso provavelmente criará um caos na lógica do seu programa.

Você percebeu que essa clonagem só é possível porque todos esses valores de estado são visíveis e podem ser facilmente copiados? Que tal um conjunto de estados encapsulados em um closure; como você clonaria esse estado?

Isso é muito mais tedioso. Essencialmente, você teria que fazer algo semelhante ao nosso método de API personalizado `forEach` anterior: fornecer uma função dentro de cada camada do closure com o privilégio de extrair/copiar os valores ocultos, criando novos closures equivalentes ao longo do caminho.

Embora isso seja teoricamente possível — outro exercício para o leitor! — é muito menos prático de implementar do que você provavelmente justificaria para qualquer programa real.

Os objetos têm uma vantagem clara quando se trata de representar o estado que precisamos clonar.

### Performance

Uma razão pela qual objetos podem ser favorecidos em relação a closures, de uma perspectiva de implementação, é que em JavaScript os objetos são frequentemente mais leves em termos de memória e até mesmo computação.

Mas tenha cuidado com isso como uma afirmação geral: há muitas coisas que você pode fazer com objetos que apagarão quaisquer ganhos de desempenho que você possa obter ao ignorar closures e mudar para rastreamento de estado baseado em objeto.

Vamos considerar um cenário com ambas as implementações. Primeiro, a implementação no estilo closure:

```js
function StudentRecord(name,major,gpa) {
    return function printStudent(){
        return `${name}, Major: ${major}, GPA: ${gpa.toFixed(1)}`;
    };
}

var student = StudentRecord( "Kyle Simpson", "CS", 4 );

// posteriormente

student();
// Kyle Simpson, Major: CS, GPA: 4.0
```

A função interna `printStudent()` fecha sobre três variáveis: `name`, `major` e `gpa`. Ela mantém esse estado sempre que transferimos uma referência para essa função -- nós a chamamos de `student()` neste exemplo.

Agora para a abordagem do objeto (e `this`):

```js
function StudentRecord(){
    return `${this.name}, Major: ${this.major}, \
GPA: ${this.gpa.toFixed(1)}`;
}

var student = StudentRecord.bind( {
    name: "Kyle Simpson",
    major: "CS",
    gpa: 4
} );

// posteriormente

student();
// Kyle Simpson, Major: CS, GPA: 4.0
```

A função `student()` -- tecnicamente chamada de "função vinculada" -- tem uma referência `this` vinculada ao literal de objeto que passamos, de modo que qualquer chamada posterior para `student()` usará esse objeto para seu `this` e, portanto, poderá acessar seu estado encapsulado.

Ambas as implementações têm o mesmo resultado: uma função com estado preservado. Mas e quanto ao desempenho; quais serão as diferenças?

**Observação:** Julgar com precisão e ação o desempenho de um trecho de código JS é um assunto muito duvidoso. Não entraremos em todos os detalhes aqui, mas recomendo que você leia *Você não conhece JS: Async & Performance*, especificamente o Capítulo 6, "Benchmarking e ajuste", para mais detalhes.

Se você estivesse escrevendo uma biblioteca que criasse um par de estado com sua função -- seja a chamada para `StudentRecord(..)` no primeiro snippet ou a chamada para `StudentRecord.bind(..)` no segundo snippet -- você provavelmente se importaria mais com o desempenho desses dois. Inspecionando o código, podemos ver que o primeiro tem que criar uma nova expressão de função a cada vez. O segundo usa `bind(..)`, que não é tão óbvio em suas implicações.

Uma maneira de pensar sobre o que `bind(..)` faz nos bastidores é que ele cria um closure sobre uma função, assim:

```js
function bind(orinFn,thisObj) {
    return function boundFn(...args) {
        return origFn.apply( thisObj, args );
    };
}

var student = bind( StudentRecord, { name: "Kyle.." } );
```

Dessa forma, parece que ambas as implementações do nosso cenário criam um closure, então o desempenho provavelmente será o mesmo.

No entanto, o utilitário interno `bind(..)` não precisa realmente criar um closure para realizar a tarefa. Ele simplesmente cria uma função e define manualmente seu `this` interno para o objeto especificado. Essa é potencialmente uma operação mais eficiente do que se fizéssemos o closure nós mesmos.

O tipo de economia de desempenho de que estamos falando aqui é minúsculo em uma operação individual. Mas se o caminho crítico da sua biblioteca estiver fazendo isso centenas ou milhares de vezes ou mais, essa economia pode aumentar rapidamente. Muitas bibliotecas -- Bluebird sendo um exemplo -- acabaram otimizando removendo closures e indo com objetos, exatamente dessa forma.

Fora do caso de uso da biblioteca, o pareamento do estado com sua função geralmente só acontece relativamente poucas vezes no caminho crítico de um aplicativo. Por outro lado, normalmente o uso da função+estado -- chamando `student()` em qualquer snippet -- é mais comum.

Se esse for o caso para alguma situação específica no seu código, você provavelmente deve se importar mais com o desempenho do último em relação ao primeiro.

Funções vinculadas historicamente tiveram um desempenho bem ruim em geral, mas recentemente foram muito mais otimizadas pelos mecanismos JS. Se você comparou essas variações alguns anos atrás, é bem possível que você obtivesse resultados diferentes repetindo o mesmo teste com os mecanismos mais recentes.

Uma função vinculada agora provavelmente terá um desempenho pelo menos tão bom, se não melhor, quanto a função fechada equivalente. Então esse é outro ponto a favor dos objetos em vez dos closures.

Só quero reiterar: essas observações de desempenho não são absolutas, e a determinação do que é melhor para um determinado cenário é muito complexa. Não aplique casualmente o que você ouviu de outros ou mesmo o que você viu em algum outro projeto anterior. Examine cuidadosamente se os objetos ou closures são apropriadamente eficientes para a tarefa.

## Sumário

A verdade deste capítulo não pode ser escrita. É preciso ler este capítulo para encontrar sua verdade.

----

Cunhar um pouco de sabedoria Zen aqui foi minha tentativa de ser inteligente. Mas você merece um resumo adequado da mensagem deste capítulo.

Objetos e fechamentos são isomórficos entre si, o que significa que eles podem ser usados ​​de forma intercambiável para representar estado e comportamento em seu programa.

A representação como um closure tem certos benefícios, como controle granular de alterações e privacidade automática. A representação como um objeto tem outros benefícios, como clonagem mais fácil de estado.

O indivíduo com pensamento crítico deve ser capaz de conceber qualquer segmento de estado e comportamento no programa com qualquer representação e escolher a representação mais apropriada para a tarefa em questão.