# Introduction
TypeScript is getting more and more popular nowadays and no wonder,
it provides nice well-known syntax, but also type safety on a level
that even some back-end languages can not beat. But I've noticed that
sometimes folks use this new shiny thing as an old one. Let's try to
demonstrate it with a simple example.

## Formulation of the problem
We are going to implement a very simple quiz app. Every question in the quiz
should have multiple answers but only one of them is correct.
A user can answer a question, navigate between questions back, forth, and by
index.
And when all the questions are answered one can finish the quiz and see his score.


We are going to take a very trivial stack: React, Redux and TypeScript.
Even though we are choosing the exact technologies,
this article is relevant for the whole front-end stack in general.

## Let's get started
So, we are going to write Redux app, the data model goes first:

```typescript
interface State {
    quiz: Quiz;
}
interface Quiz {
    questions: Question[];
    current: number;
}
interface Question {
    text: string;
    answers: string[];
    correctAnswer: number;
    selectedAnser?: number;
}
```

So far so good, some actions next:

```typescript
const NEXT_QUESTION = 'NEXT_QUESTION';
const PREVIOUS_QUESTION = 'PREVIOUS_QUESTION';
const GO_TO_QUESTION = 'GO_TO_QUESTION';
const ANSWER_CURRENT_QUESTION = 'ANSWER_CURRENT_QUESTION';

interface NextQuestionAction {
    type: typeof NEXT_QUESTION;
}
interface PreviousQuestionAction {
    type: typeof PREVIOUS_QUESTION;
}
interface GoToQuestionAction {
    type: typeof GO_TO_QUESTION;
    n: number;
}
interface AnswerCurrentQuestionAction {
    type: typeof ANSWER_CURRENT_QUESTION;
    n: number;
}
type Action =
    | NextQuestionAction
    | PreviousQuestionAction
    | GoToQuestionAction
    | AnswerCurrentQuestionAction 
```

And the most interesting part, reducer. Redux author strongly suggests using
immutable data transformations inside a reducer and I would highly suggest
following this rule rigorously, otherwise, you gonna have very weird bugs
and bad performance all over the place. But luckily we have the spread operator,
it's not going to be a problem, right?

```typescript
function reducer(state: State = defaultState, action: Action): State {
    switch(action.type) {
        case NEXT_QUESTION: {
            if (state.quiz.questions.length < state.quiz.currentQueston + 1) {
                return {...state, quiz: {...state.quiz, current: state.quiz.current + 1 }};
            } else {
                return state;
            }
...
```

A bit verbose but nothing too fancy. Almost the same logic will be for
`PREVIOUS_QUESTION` and `GOTO_QUESTION`, just don't forget to verify an
array boundary. A little funny it looks for `ANSWER_CURRENT_QUESTION`

```typescript
case ANSWER_CURRENT_QUESTION: {
    // First we need to update only CURRENT question in the list of questions,
    // that means array should be brand new, but all elements except one should be the same
    const questions = state.questions.map(question => {
        if (state.quiz.current == action.n) {
            // We found it, current question is here!
            // But is the answer number valid
            if (action.n >= question.answers.length)
                // I don't know what to do here.. maybe throwing an exception is not such a bad idea
                throw new Error("There is no answers with such index");
            // Ok, index is valid, now update question
            return { ...question, selectedAnswer: action.n };
        } else {
            return question;
        }
    });
    // We are not done yet, we need to also update the state and the quiz
    return { ...state, quiz: { ...state.quiz, questions } };
}

```

WOW...This is too much even for me. So much ceremony just for nothing,
and we are only 3 levels deep, what if we go deeper... No wonder some
clever guys invented immutable.js, mori and Ramda (which is actually quite
good). But there is also one new and already quite popular player,
[Immer](https://github.com/immerjs/immer).

> Create the next immutable state tree by simply modifying the current tree
> Winner of the "Breakthrough of the year" React open source award and
> "Most impactful contribution" JavaScript open source award in 2019

Sounds like exactly what we need, maybe it saves our lives against boredom?
Let's try:

```typescript
import produce from 'immer';

...
case NEXT_QUESTION: {
    if (state.quiz.questions.length < state.quiz.currentQueston + 1) {
        // Actually not that much of a difference here
        return produce(state, draft => {
            draft.quiz.current++;
        });
    } else {
        return state;
    }
}
case ANSWER_CURRENT_QUESTION: {
    produce(state, draft => {
        const currentQuestion = draft.quiz.questions[draft.quiz.current];
        if (action.n >= currentQuestion.answers.length)
            throw new Error("There is no answers with such index");
        currentQuestion.selectedAnswer = action.n;
    })
}       
```

Not bad at all! Very impressive, isn't it? We are done, problem is solved,
just install immer and go to prod.

But let's try to do a step back, look at the problem from another point of view.
I'd ask one question: what reducer knows? Turns out it knows quite a lot of things:
- It knows that there is a state and this state has a quiz field.
- Quiz field is a record that has a list of questions and the current question
as a number
- Each question has some text and some answers inside (which are just strings).
- Each question may or may not be answered.
- It knows how to navigate between questions, how to answer question

Actually, it knows everything, every tiny detail of our app. We can extract some
helper functions, move them somewhere (try to find them after that).
And while our app grows, reducer knows more and more. We just implemented an
antipattern "the God Object", but in our case, it's "the God Function".
Looks like we missed the forest for the tree.

So what can be done here? How the ideal reducer should look like? In my fantasy
land with pink unicorns and rainbow the ideal API would look kinda like this:

```typescript
case NEXT_QUESTION:
    return { ...state, quiz: state.quiz.next() };
case ANSWER_CURRENT_QUESTION:
    return { ...state, quiz: state.quiz.answerCurrentQuestion(action.n) };
```

Can we implement that? Yes, we can! Actually, we can even do it in two ways. 

# Solution 1: Hello OOP.

Our app has several invariants that we want to enforce:

- Quiz has several questions
- Initially, the first question is current
- Every question has several answers
- Only one of them is correct
- A question may or may not have selected answer
- We can finish quiz only when all questions are answered

But in general, how can we enforce an invariant. I'm aware of 3 ways to do so:

1. Verifying them manually (you don't want to do that)
2. Via tests
3. Via type system

I strongly prefer the third way against all others (testing is also fine,
but as an addition, not alone).
What can go wrong with our initial data structure?

```typescript
interface Quiz {
    questions: Question[];
    current: number;
}
```

Quite a lot actually. Does it make sense to have a quiz with zero elements?
What if `current` is out of range for `questions`?
Here's the first attempt to make it better:

```typescript
class Quiz {
    private readonly prevL: Array<Question>;
    private readonly current: Question;
    private readonly nextL: Array<Question>;
}
```

Every field is `readonly` to enforce immutability. Here we have two lists
for previous and next questions and one current. Having next and previous
arrays empty is totally valid, but current should be there anyway.
Everything is private here and this is very important.
Nobody should know what is inside our class, that way we can change our
inner implementation and API still stays the same.
But how to create an instance of our class?

```typescript
// Constructor here is private, this is very important
// it's only for us and should not be used outside
private constructor(
    prevL: Array<Question>,
    current: Question,
    nextL: Array<Question>,
) {
    this.prevL = prevL;
    this.current = current;
    this.nextL = nextL;
}

// This method is public and it's an actual way to create
// and instance. The only parameter here is the non-empty
// list of questions. The first element immediately gets current
// all others go to next array and the prev array is left empty
public static init([first, ...rest]: [Question, ...Question[]]): Quiz {
    return new Quiz([], first, rest);
}
```
So, how can we navigate back and forth between our questions?
```typescript
// Very useful function can be used from outside,
// and also as part of next and previous
public gotoNth(n: number) {
    if (this.prevL.length == n) return this;
    const lst = this.toArray();
    // Still need to handle boudaries here
    if (lst.length <= n)
        throw new Error(`There is no question with index ${n}`);

    const prevL = lst.slice(0, n);
    const current = lst[n];
    const nextL = lst.slice(n + 1, lst.length);
    // Just create a new quiz from the current one
    // using our private constructor
    return new Quiz(prevL, current, nextL);
}

public hasNext(): boolean {
    return this.nextL.length != 0;
}

public next(): Quiz {
    // If we already at the end, just stay where you are
    if (!this.hasNext()) return this;

    return this.gotoNth(this.prevL.length + 1);
}
```

Notice that most functions return `Quiz` again.
This way we can safely compose our operations via dot:

```typescript
// One step forward
quiz.next();
// One back, two forward
quiz.previous().next().next();
// Answer current and go to next
quiz.answerCurrentQuestion(2).next();
```

I believe you can do the rest all by yourself. Here the method signatures:

```typescript
public static init([first, ...rest]: [Question, ...Question[]]): Quiz;
public getCurrent(): Question;
public currentNumber(): number;
public hasNext(): boolean;
public next(): Quiz;
public hasPrevious(): boolean;
public previous(): Quiz;
public gotoNth(n: number);
public size(): number;
public answerCurrentQuestion(n: number): Quiz;
public fullyAnswered(): boolean;
public finish(): Quiz;
public isFinished(): boolean;
public getScore(): [number, number];
private toArray(): Array<Question>;
```

Here an [example](https://github.com/denistakeda/modulas/blob/master/src/OOP/Quiz.ts) if you are stuck.
Question class much simpler, but follows the same rules:

```typescript
class Question {
    // Don't see anything against making them public,
    // but getters may be written instead
    public readonly text: string;
    public readonly answers: Array<string>;
    private readonly correctAnswer: number;
    // May or may not be answered
    private readonly selectedAnswer?: number;

    public constructor(
        text: string,
        answers: Array<string>,
        correctAnswer: number,
        selectedAnswer?: number
    ) {
        this.text = text;
        this.answers = answers;
        this.correctAnswer = correctAnswer;
        this.selectedAnswer = selectedAnswer;
    }

    public answer(n: number) {
        if (n >= this.answers.length)
            throw new Error(`There are no answers with number ${n}`);

        return new Question(this.text, this.answers, this.correctAnswer, n);
    }

    public isAnswered(): boolean {
        return typeof this.selectedAnswer != 'undefined';
    }

    public isAnsweredCorrectly(): boolean {
        if (typeof this.selectedAnswer == 'undefined') return false;

        return this.selectedAnswer == this.correctAnswer;
    }

    public isSelectedAnswer(n: number) {
        return n == this.selectedAnswer;
    }
}
```

# Is this better?

I'm not a big fan of OOP paradigm, but this solution is definitely better than
what we had initially. If we want to add new functionality to the quiz, there
is only one place we can put it, as well as when we are searching for some
quiz-related functionality. What does reducer know about our app?
It knows that there is a quiz, it can be iterated back and forth and it can
be answered (so no consideration about shape, Quiz is a black box for reducer).
Notice it knows nothing about the question at all.
Also, changes are quite easy to add now, refactoring is also getting very
simple,
just add tests for the public API and do whatever you want inside.

If you want some more extended example with all the glue code altogether,
I created and [example](https://github.com/denistakeda/modulas) repo for you.

In the next article, I'll show how how to go even further and implement the
same logic in the functional paradigm.
