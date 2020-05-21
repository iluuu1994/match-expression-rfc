# PHP RFC: Match expression v2

* Date: 2020-05-16
* Author: Ilija Tovilo, tovilo.ilija@gmail.com
* Status: Under discussion
* Target Version: PHP 8.0
* Implementation: [https://github.com/php/php-src/pull/5371](https://github.com/php/php-src/pull/5371)
* Supersedes: [https://wiki.php.net/rfc/match_expression](https://wiki.php.net/rfc/match_expression)

## Proposal

This RFC proposes adding a new `match` expression that is similar to `switch` but with safer semantics and the ability to return values.

[From the Doctrine query parser](https://github.com/doctrine/orm/blob/72bc09926df1ff71697f4cc2e478cf52f0aa30d8/lib/Doctrine/ORM/Query/Parser.php#L816):

```php
// Before
switch ($this->lexer->lookahead['type']) {
    case Lexer::T_SELECT:
        $statement = $this->SelectStatement();
        break;

    case Lexer::T_UPDATE:
        $statement = $this->UpdateStatement();
        break;

    case Lexer::T_DELETE:
        $statement = $this->DeleteStatement();
        break;

    default:
        $this->syntaxError('SELECT, UPDATE or DELETE');
        break;
}

// After
$statement = match ($this->lexer->lookahead['type']) {
    Lexer::T_SELECT => $this->SelectStatement(),
    Lexer::T_UPDATE => $this->UpdateStatement(),
    Lexer::T_DELETE => $this->DeleteStatement(),
    default => $this->syntaxError('SELECT, UPDATE or DELETE'),
};
```

## Differences to switch

### Return value

It is very common that the `switch` produces some value that is used afterwards.

```php
switch (1) {
    case 0:
        $result = 'Foo';
        break;
    case 1:
        $result = 'Bar';
        break;
    case 2:
        $result = 'Baz';
        break;
}

echo $result;
//> Bar
```

It is easy to forget assigning `$result` in one of the cases. It is also visually unintuitive to find `$result` declared in a deeper nested scope. `match` is an expression that evaluates to the result of the executed arm. This removes a lot of boilerplate and makes it impossible to forget assigning a value in an arm.

```php
echo match (1) {
    0 => 'Foo',
    1 => 'Bar',
    2 => 'Baz',
};
//> Bar
```

### No type coercion

The `switch` statement loosely compares (`==`) the given value to the case values. This can lead to some very surprising results.

```php
switch ('foo') {
    case 0:
      $result = "Oh no!\n";
      break;
    case 'foo':
      $result = "This is what I expected\n";
      break;
}
echo $result;
//> Oh no!
```

The `match` expression uses strict comparison (`===`) instead. The comparison is strict regardless of `strict_types`.

```php
echo match ('foo') {
    0 => "Oh no!\n",
    'foo' => "This is what I expected\n",
};
//> This is what I expected
```

### No fallthrough

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

The `match` expression resolves this problem by adding an implicit `break` after every arm.

```php
match ($pressedKey) {
    Key::RETURN_ => save(),
    Key::DELETE => delete(),
};
```

Multiple conditions can be comma-separated to execute the same block of code.

```php
echo match ($x) {
    1, 2 => 'Same for 1 and 2',
    3, 4 => 'Same for 3 and 4',
};
```

### Exhaustiveness

Another large source of bugs is not handling all the possible cases supplied to the `switch` statement.

```php
switch ($operator) {
    case BinaryOperator::ADD:
        $result = $lhs + $rhs;
        break;
}

// Forgot to handle BinaryOperator::SUBTRACT
```

This will go unnoticed until the program crashes in a weird way, causes strange behavior or even worse becomes a security hole. Many languages can check if all the cases are handled at compile time or force you to write a `default` case if they can't. For a dynamic language like PHP the only alternative is throwing an error at runtime. This is exactly what the `match` expression does. It throws an `UnhandledMatchError` if the condition isn't met for any of the arms.

```php
$result = match ($operator) {
    BinaryOperator::ADD => $lhs + $rhs,
};

// Throws when $operator is BinaryOperator::SUBTRACT so we catch the mistake early on
```

## Miscellaneous

### Arbitrary expressions

A match condition can be any arbitrary expression. Analogous to `switch` each condition will be checked from top to bottom until the first one matches. If a condition matches the remaining conditions won't be evaluated.

```php
$result = match ($x) {
    foo() => ...,
    $this->bar() => ..., // bar() isn't called if foo() matched with $x
    $this->baz => ...,
    // etc.
};
```

## Future scope

### Blocks

In this RFC the body of a match arm must be an expression. Blocks for match and arrow functions will be discussed in a separate RFC.

### Pattern matching

[I have experimented with pattern matching](https://github.com/php/php-src/compare/master...iluuu1994:pattern-matching) and decided not to include it in this RFC. Pattern matching is a complex topic and requires a lot of thought. Each pattern should be discussed in detail in a separate RFC.

### Allow dropping (true)

```php
$result = match { ... };
// Equivalent to
$result = match (true) { ... };
```

## Backward Incompatible Changes

`match` was added as a keyword (`reserved_non_modifiers`). This means it can't be used in the following contexts anymore:

* namespaces
* class names
* function names
* global constants

Note that it will continue to work in method names and class constants.

## Vote

Voting started 2020-xx-xx and closes 2020-xx-xx.
