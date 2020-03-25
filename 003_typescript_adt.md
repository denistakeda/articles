# Functional Typescript: Algebraic data types

In my [previous](/001_modularized_frontend.md) article I've shown how to
split application by modules in an OOP way and promised to show FP way
also. But functional programming technicks are not so widely explained
in documentation and people in general are less aware how to do it
(just because it's less popular). I decided first to show some general
FP technicks and then combine them together to create an app.

In this article I'll show how to implement algebraic data types in
TypeScript (they are called
[discriminated
unions](https://www.typescriptlang.org/docs/handbook/advanced-types.html#discriminated-unions)
here). Quite a usefull instrument to model our domain model and 
underesimated in favor of classes.

## Modules instead of classes

In the OOP world we have one has quite a deligent way to encapsulate the
inner details of data manipulations, the are public/protected/private 
modificators. Using this we can easy hide implementation detailes under
a fence of classes.

First time I tried to use functional programming paradigm in Typescript,
I did't know how to do this sort of thing in the right way, but turns out
there is nothing too fancy here, we already have everything we need for
that.

In languages like Haskell or Elm people often use modules for similar purposes.
The idea is quite simple here, module is a file, often centralized around a
single type. Managing the access rights is done by exporting or not
exporting functions and types. Funcion or type that is exported considered
public.
Here an example:

```typescript
// interface is exported, so it is public
export interface Question { 
    id: number;
    text: string;
    answers: Array<string>;
    correctAnswer: number;
}

// public function
export function isAnsweredCorrectly(question: Question): boolean {
 ...
}

// function is not exported, so it is private
function _isSelectedAnswer(question: Question): boolean {
 ...
}

```

Some people tend to use underscore before the function to emphasize its privaty.

## Discriminant

There are two ways you can model data, with open sets (classes and polymorphism)
and closed sets (algebraic data types). If you know beforehead the number of
possibilities, then it's better to enumerate them all. To do so ADT usually
used.

In Typescript ADT are also allowed even though a bit limited (you can only use
them with records). The creation includes three steps.

### Step 1: string constants

```typescript
const UNANSWERED = 'UNANSWERED';
const ANSWERED = 'ANSWERED';
```

### Step 2: interfaces

```typescript
interface Unanswered {
    // discriminant property. May have any name, but should be consistent
    // across interfaces
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
```

### Step 3: combine interfaces into on type

```typescript
type Question = Unanswered | Answered;
```

In such a way our question can be in two completely different states, answered
and not and we have different sets of fields for each state. Morower, Typescript
compiler can statically check in which state a question is now and allow for
acces only properties from this state.

## Usage

Let's have a look at the example usage:

```typescript
// first we need to create a constant with type question
const question: Question = {
    kind: UNANSWERED,
    
    text: 'What is the best programming language?',
    answers: ['Typescript', 'JavaScript', 'Haskell'],
    correctAnswer: 0,
}

// then we can define some operations:
function answerQuestion(question: Question, selectedAnswer: number): Question {
    switch (question.kind) { // switch against discriminant property
        case UNANSWERED: 
            // in this branch compiler already knows that question is of
            // type Unanswered. It does not allow to use question.selectedAnswer
            // for example. Try it yourself.
            return {
                kind: ANSWERED,
                selectedAnswer: selectedAnswer,
                isCorrect: selectedAnswer === question.correctAnswer,
            };
        case ANSWERED:
            // in this branch the question is already answered, it doesn't make
            // sense to answer it again. So we can just return the same question
            return question;
    }
}
```

## Conclusion

Discriminant union types (or Algebraic Data Types) is an awesome instrument to
model a closed set of types. It's fully supported by TypeScript compiler
and a good fit to describe some known set of types for a single value. In next
articles I'll show how to use this construction in a practical application.
