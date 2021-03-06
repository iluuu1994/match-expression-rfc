# match expression RFC

* Date: 2020-04-12
* Author: Ilija Tovilo, tovilo.ilija@gmail.com
* Status: Voting
* Target Version: PHP 8.0
* Implementation: https://github.com/php/php-src/pull/5371
* Supersedes: https://wiki.php.net/rfc/switch_expression

## Proposal

The `switch` statement is a fundamental control structure in almost every programming language. Unfortunately, in PHP it has some long-standing issues that make it hard to use correctly, namely:

* Type coercion
* No return value
* Fallthrough
* Inexhaustiveness

This RFC proposes a new control structure called `match` to resolve these issues.

```php
match ($condition) {
    1 => {
        foo();
        bar();
    },
    2 => baz(),
}

$expressionResult = match ($condition) {
    1, 2 => foo(),
    3, 4 => bar(),
    default => baz(),
};
```

## Issues

We're going to take a look at each issue and how the new `match` expression resolves them.

### Type coercion

The `switch` statement loosely compares (`==`) the given value to the case values. This can lead to some very surprising results.

```php
switch ('foo') {
    case 0:
      echo "Oh no!\n";
      break;
}
```

The `match` expression uses strict comparison (`===`) instead. The comparison is strict regardless of `strict_types`.

```php
match ('foo') {
    0 => {
        echo "Never reached\n";
    },
}
```

### No return value

It is very common that the `switch` produces some value that is used afterwards.

```php
switch (1) {
    case 0:
        $y = 'Foo';
        break;
    case 1:
        $y = 'Bar';
        break;
    case 2:
        $y = 'Baz';
        break;
}

echo $y;
//> Bar
```

It is easy to forget assigning `$y` in one of the cases. It is also visually unintuitive to find `$y` declared in a deeper nested scope. `match` is an expression that evaluates to the result of the executed arm. This removes a lot of boilerplate and makes it impossible to forget assigning a value in an arm.

```php
echo match (1) {
    0 => 'Foo',
    1 => 'Bar',
    2 => 'Baz',
};
//> Bar
```

### Fallthrough

The `switch` fallthrough has been a large source of bugs in many languages. Each `case` must explicitly `break` out of the `switch` statement or the execution will continue into the next `case` even if the condition is not met.

```php
switch ($pressedKey) {
    case Key::RETURN_:
        save();
        // Oops, forgot the break
    case Key::DELETE:
        delete();
        break;
}
```

This was intended to be a feature so that multiple conditions can execute the same block of code. It is often hard to understand if the missing `break` was the authors intention or a mistake.

```php
switch ($x) {
    case 1:
    case 2:
        // Same for 1 and 2
        break;
    case 3:
        // Only 3
    case 4:
        // Same for 3 and 4
        break;
}
```

The `match` expression resolves this problem by adding an implicit `break` after every arm. Multiple conditions can be comma-separated to execute the same block of code. There's no way to achieve the same result as 3 and 4 in the example above without an additional `if` statement. This is a little bit more verbose but makes the intention very obvious.

```php
match ($x) {
    1, 2 => {
        // Same for 1 and 2
    },
    3, 4 => {
        if ($x === 3) {
            // Only 3
        }
        // Same for 3 and 4
    },
}
```

### Inexhaustiveness

Another large source of bugs is not handling all the possible cases supplied to the `switch` statement.

```php
switch ($configuration) {
    case Config::FOO:
        // ...
        break;
    case Config::BAR:
        // ...
        break;
}
```

This will go unnoticed until the program crashes in a weird way, causes strange behavior or even worse becomes a security hole. Many languages can check if all the cases are handled at compile time or force you to write a `default` case if they can't. For a dynamic language like PHP the only alternative is throwing an error. This is exactly what the `match` expression does. It throws an `UnhandledMatchError` if the condition isn't met for any of the arms.

```php
match ($x) {
    1 => ...,
    2 => ...,
}

// $x can never be 3
```

## Blocks

Sometimes passing a single expression to a match arm isn't enough, either because you need to use a statement or the code is just too long for a single expression. In those cases you can pass a block to the arm.

```php
match ($x) {
    0 => {
        foo();
        bar();
        baz();
    },
}
```

Originally this RFC included a way to return a value from a block by omitting the semicolon of the last expression. This syntax is borrowed from Rust (https://doc.rust-lang.org/reference/expressions/block-expr.html). Due to [memory management difficulties](https://github.com/php/php-src/pull/5448) and a lot of negative feedback on the syntax this is no longer a part of this proposal and will be discussed in a separate RFC.

```php
// Original proposal
$y = match ($x) {
    0 => {
        foo();
        bar();
        baz() // This value is returned
    },
};

// Alternative syntax, <=
$y = match ($x) {
    0 => {
        foo();
        bar();
        <= baz();
    },
};

// Alternative syntax, separate keyword
$y = match ($x) {
    0 => {
        foo();
        bar();
        pass baz();
    },
};

// Alternative syntax, automatically return last expression regardless of semicolon
$y = match ($x) {
    0 => {
        foo();
        bar();
        baz();
    },
};
```

For the time being using blocks in match expressions that use the return value in any way results in a compilation error:

```php
$y = match ($x) {
    0 => {},
};
//> Match that is not used as a statement can't contain blocks

foo(match ($x) {
    0 => {},
});
//> Match that is not used as a statement can't contain blocks

1 + match ($x) {
    0 => {},
};
//> Match that is not used as a statement can't contain blocks

//etc.

// Only allowed form
match ($x) {
    0 => {},
}
```

## Optional semicolon for match in statement form

When using `match` as part of some other expression it is necessary to terminate the statement with a semicolon.

```php
$x = match ($y) { ... };
```

The same would usually be true if the `match` expression were used as a standalone expression.

```php
match ($y) {
    ...
};
```

However, to make the `match` expression more similar to other statements like `if` and `switch` we could allow dropping the semicolon in this case only.

```php
match ($y) {
    ...
}
```

This introduces an ambiguity with the `+` and `-` unary operators.

```php
match ($y) { ... }
-1;

// Could be parsed as

// 1
match ($y) { ... };
-1;

// 2
match ($y) { ... } - 1;
```

A `match` that appears as the first element of a statement would always be parsed as option 1 because there are no legitimate use cases for binary operations at a statement level. All other cases work as expected.

```php
// These work fine
$x = match ($y) { ... } - 1;
foo(match ($y) { ... } - 1);
$x[] = fn($y) => match ($y) { ... };
// etc.
```

This is also how Rust solves this ambiguity (https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=31b5bca165323726e7aa8d14f62f94d5).

```rust
match true { _ => () } - 1;

// warning: unused unary operation that must be used
//  --> src/main.rs:2:28
//   |
// 2 |     match true { _ => () } - 1;
//   |                            ^^^
//   |
```

Because there was some controversy around this feature it was moved to a secondary vote.

## Allow dropping (true) condition

It has been suggested to make no condition equivalent to `(true)`.

```php
match {
    $age >= 30 => {},
    $age >= 20 => {},
    $age >= 10 => {},
    default => {},
}

// Equivalent to

match (true) {
    $age >= 30 => {},
    $age >= 20 => {},
    $age >= 10 => {},
    default => {},
}
```

The keyword `match` could be a bit misleading here. A potential gotcha is passing truthy values to the match which will not work as intended. But of course this issue remains regardless of dropping `(true)` or not.

```php
match {
    preg_match(...) => {}, // preg_match returns 1 which is NOT identical (===) to true
}
```

Because I have no strong opinion on this it will be moved to a secondary vote.

## Miscellaneous

### Arbitrary expressions

A match condition can be any arbitrary expression. Analogous to `switch` each condition will be checked from top to bottom until the first one matches. If a condition matches the remaining conditions won't be evaluated.

```php
match ($x) {
    foo() => ...,
    $this->bar() => ..., // bar() isn't called if foo() matched with $x
    $this->baz => ...,
    // etc.
}
```

### break/continue

Just like with the switch you can use `break` to break out of the executed arm.

```php
match ($x) {
    $y => {
        if ($condition) {
            break;
        }

        // Not executed if $condition is true
    },
}
```

Unlike the switch using `continue` targeting the `match` expression will trigger a compilation error.

```php
match ($i) {
    default => {
        continue;
    },
}
//> Fatal error: "continue" targeting match is disallowed. Did you mean to use "break" or "continue 2"?
```

### goto

Like with the `switch` you can use `goto` to jump out of `match` expressions.

```php
match ($a) {
    1 => {
        match ($b) {
            2 => {
                goto end_of_match;
            },
        }
    },
}

end_of_match:
```

It is not allowed to jump into match expressions.

```php
goto match_arm;

match ($b) {
    1 => {
        match_arm:
    },
}

//> Fatal error: 'goto' into loop, switch or match is disallowed
```

### return

`return` behaves the same as in any other context. It will return from the function.

```php
function foo($x) {
    match ($x) {
        1 => {
            return;
        },
    }

    // Not executed if $x is 1
}
```

## Future scope

### Block expressions

As mentioned above block expressions will be discussed in a separate RFC. We'll also use this opportunity to think about blocks in arrow functions.

### Pattern matching

[I have experimented with pattern matching](https://github.com/php/php-src/compare/master...iluuu1994:pattern-matching) and decided not to include it in this RFC. Pattern matching is a complex topic and requires a lot of thought. Each pattern should be discussed in detail in a separate RFC.

```php
// With pattern matching
match ($value) {
    let $a => ..., // Identifer pattern
    let 'foo' => ..., // Scalar pattern
    let 0..<10 => ..., // Range pattern
    let is string => ..., // Type pattern
    let [1, 2, $c] => ..., // Array pattern
    let Foo { foo: 1, getBar(): 2 } => ..., // Object pattern
    let $str @ is string if $str !== '' => ..., // Guard

    // Algebraic data types if we ever get them
    let Ast::BinaryExpr($lhs, '+', $rhs) => ...,
}

// Without pattern matching
match (true) {
    true => $value ..., // Identifier pattern
    'foo' => ..., // Scalar pattern
    $value >= 0 && $value < 10 => ..., // Range pattern
    is_string($value) => ..., // Type pattern
    count($value) === 3
        && isset($value[0]) && $value[0] === 1
        && isset($value[1]) && $value[1] === 2
        && isset($value[2]) => $value[2] ..., // Array pattern
    $value instanceof Foo
        && $value->foo === 1
        && $value->getBar() === 2 => ..., // Object pattern
    is_string($str) && $str !== '' => ..., // Guard
}
```

### Explicit fallthrough

Some people have suggested allowing explicit fallthrough to the next arm. This is, however, not a part of this RFC.

```php
match ($x) {
    1 => {
        foo();
        fallthrough;
    },
    2 => {
        bar();
    },
}

// 1 calls foo() and bar()
// 2 only calls bar()
```

This would require a few sanity checks with pattern matching.

```php
match ($x) {
    $a => { fallthrough; },
    $b => { /* $b is undefined */ },
}
```

## "Why don't you just use x"

### if statements

```php
if ($x === 1) {
    $y = ...;
} elseif ($x === 2) {
    $y = ...;
} elseif ($x === 3) {
    $y = ...;
}
```

Needless to say this is incredibly verbose and there's a lot of repetition. It also can't make use of the jumptable optimization. You must also not forget to write an `else` statement to catch unexpected values.

### Hash maps

```php
$y = [
    1 => ...,
    2 => ...,
][$x];
```

This code will execute every single "arm", not just the one that is finally returned. It will also build a hash map in memory every time it is executed. And again, you must not forget to handle unexpected values.

### Nested ternary operators

```php
$y = $x === 1 ? ...
  : ($x === 2 ? ...
  : ($x === 3 ? ...
  : 0));
```

The parentheses make it hard to read and it's easy to make mistakes and there is no jumptable optimization. Adding more cases will make the situation worse.

### Backward Incompatible Changes

`match` was added as a keyword (`reserved_non_modifiers`). This means it can't be used in the following contexts anymore:

* namespaces
* class names
* function names
* global constants

Note that it will continue to work in method names and class constants.
