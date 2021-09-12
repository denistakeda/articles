# Functional Typescript: Opaque Types

Objective oriented programming paradigm provides a wide set of encapsulation
mechanisms. It has public/private/protected class-level modificators for
fields and inheritance which create a high level of encapsulation. But what
is the notion of encapsulation? Why do we want to have it in general? There
are a couple of reason why do we might to have this trick in our toolbelt:

- We want to hide wrap some inner data structure to provide efficient algorithm
via API, which we don't want any strangers to modify
- We want to hide some implementation details to be able to change them later
and not break anyone's code
- We want to disallow adding any functionality to our datatype anywhere except
one exact place: wrapping class.

The third option can actually be easily broken via inheritance (this is one of
the reasons why inheritance considered as an antipattern).

But this is OOP, in functional programming there is no classes and therefore
no visibility modificators. Data types are usually defined top-level and
completely separated from functions. So should we give up encapsulation in FP?

Actually no. There is diligent pattern to achieve the same level of encapsulation
in functional programming, it's called *Opaque Types*.

## What is *Opaque Types*?

There is no special keyword or some special modificator to create this sort of
types. *Opaque Type* is just an algebraic type that exposes its definition 
but hides type constructors. That's it, nothing too fancy. Still don't get it
:-)? Then it's time for examples.


## Implementation

*Opaque Types* are widely used in many strongly typed functional languages, like
Haskell or Elm, but for some reason I've never seen even a mention them
in TypeScript context. This is unfortunate, because they can be easily
implemented and very usefull there as well. Let's try to do it by example.

### Implementing ADT

In my [previous](/003_typescript_adt.md) article I explained *Algebraic Data
Types* in TypeScript in all details, please read it first if you did not
do it already, we will do an extensive usage of this concept. So, first we
need to create and algebraic data type.

In this example we are going to implement Question type for some quiz. Question
can be in two states, answered and not. If question is not answered yet, then
we want to have the full information, like question text, list of available
answers and the number of a correct answer as well. If question is already 
answered then we don't need most of its information anymore, we need only
the selected answer number and is this correct or not.

```typescript
const UNANSWERED = 'UNANSWERED';
const ANSWERED = 'ANSWERED';

interface Unanswered {
    kind: typeof UNANSWERED;
    
    text: string;
    answers: Array<string>;
    correctAnswer: number;
}

interface Answered {
    kind: typeof ANSWERED;
    
    selectedAnswer: number;
    isCorrect: boolean;
}

type Question = Unanswered | Answered;
```

### Restrict visibility rules

Ok, we have the type now, but this type if very open now. Anyone can create
a question in any of its states. Like, for example what does it mean to
have an answered question without having unanswered before? What if one
creates an unanswered question where `correctAnswer` number is bigger than
the length of `answers` array? All sort of inconsistency may happen and
right now we can not prohibit this.

A couple of simple questions arises here:

- Do we want some data of `Question` type in the external code? *Yes, we want*
- Do we want an external code to be able to create an any type of question
in any state? *No, we don't*

If the question were class, most likely it would have a private constructor
and some public construction function, that allows only defined set of
operations, but we don't have public/private modificators here, so how
to achieve the same level of encapsulation? We can do it by using `export`
keyword

```typescript
// type constants are not exported, so they can not be used from the
// external code
const UNANSWERED = 'UNANSWERED';
const ANSWERED = 'ANSWERED';

// as well as type interfaces, nobody except our own module can use them
// in other languages these are usually called type constructors
interface Unanswered {
    kind: typeof UNANSWERED;
    
    text: string;
    answers: Array<string>;
    correctAnswer: number;
}

interface Answered {
    kind: typeof ANSWERED;
    
    selectedAnswer: number;
    isCorrect: boolean;
}

// but type itself should be exposed to the outside world
export type Question = Unanswered | Answered;
```

### Constructor functions

Ok, now we have only type, what can we do with it from the external code? 
We can define a variable of this type:

```typescript
import { Question } as Question from 'question';

let question: Question;
```

We can define function that accepts or returns `Question`:

```typescript
function iAcceptQuestion(question: Question): void {...}
function iReturnQuestion(): Question {..}
```

And actually nothing more. But we miss a way to create an instance of our type.
The classical way to do that is to create a constructor function inside our
module. In this constructor we can do all sort of verifications and
confirmations and create only consistent state:

```typescript
export function create(text: string, answers: string[], correctAnswer: number): Question {
    if (answers.length < 2)
        throw new Error('It doesn\'t make sense to have a question with less than two answers');
    if (correctAnswer < 0 || correctAnswer >= answers.length)
        throw new Error('The number of correct answer should be in the range of answers');
        
    return { kind: UNANSWERED, text, answers, correctAnswer };
}
```

All these exceptions look a bit non-FP, and they are also really hard to
compose. Instead, we can use a `Result` type, but this is the material for
the full article, so will wait till later.

After we create our constructor function it would be nice to have some 
workflow functions as well. For example, in this case we need a way to
answer a question.

```typescript
export function answer(selectedAnswer: number, question: Question): Question {
    switch (question.kind) {
        case UNANSWERED: { // since now compiler knows that question is of Unanswered type
            if (selectedAnswer < 0 || selectedAnswer >= question.answers.length)
                throw new Error('The number of selected answer should be in the range of answers');
            return {
                kind: ANSWERED,
                isCorrect: selectedAnswer === question.correctAnswer,
                selectedAnswer,
            };
        }
        case ANSWERED:
            // question is already answered, answer the same question twice
            // is not allowed in our system, so just return question
            return question;
    }
}
```

Our implementation and the structure of the type is completely hidden now,
therefore we don't have a way to get any information from it, we can not
read any field. But we still need to know something, for example is our
question answered or not, is it answered correctly. We have only one way
to implement this and only one place to put the implementation: the question
module

```typescript
export function isAnswered(question: Question): boolean {
    return question.kind === ANSWERED;
}

export function isAnsweredCorrectrly(question: Question): boolean {
    switch (question.kind) {
        case UNANSWERED:
            // Something that is not answered is not answered correctly either
            return false;
        case ANSWERED:
            return question.isCorrect;
    }
}
```

### Usage

Using *Opaque Types* is quite simple. Just import the module and use functions:

```typescript
import * as Q from './question';

// Creation of an instance
const question: Q.Question = Q.create(
    "What is your favorite prgramming language?",
    ["JavaScript", "TypeScript", "Elm"],
    1,
);

Q.isAnswered(question); // => false
Q.isAnsweredCorrectrly(question); // => false

// Answer created question
const answeredQuestion: Q.Question = Q.answer(1, question);

Q.isAnswered(answeredQuestion); // => true
Q.isAnswered(answeredQuestion); // => true
```

Please do not forget that all data types should be immutable. We'll have
one more article about immutability, but for now just try to never mutate
anything.

## Why is it helpful

The *Opaque Types* give us the same level of encapsulation, safety and
consistency as classes from OOP, but the cost is quite lower, we don't
have implicit `this` state, which might be source of all sorts of disaster.
Also opaque types do not own your data, your code own data (as you can see
in the example above), module for opaque types just provide a useful 
interface to work with your data, which might be very important in some
cases.

## Downsides

Every poverfull tool has its own downsides that come with it. 
*Opaque Types* is not an exception. You should be careful when choosing
to make a type opaque. First of all, implementation will be hidden,
which is very good in some cases and very bad in others. If you find
yourself writing getters and setter for most fields in your type
(as we did with `isAnswered` and `isAnsweredCorrectly`), then most
likely this type should NOT be opaque.

Also, by making type opaque, you close the set of functions that
can operate on internal structure of the type. This is much more
strict constraint than OOP class, because you can extend class,
but you can never extend opaque type. Functions that are written
in the module are the only functions that can operate on internal
structure of your type.

## Wraping Up

In this article I explained how can we encapsulate type inside the
module by using *Opaque Type*, we've looked at the example usage
and detailed implementation. This will be very helpful when we
get back to implement our [quiz app](/001_modularized_frontend.md) later.
In the next article I show what is immutability, why is this important,
how to use it correctly, what are persistent data transformations and
persistent data types. This will be the last step before combining
everything together to create a functional app.
