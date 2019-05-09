---
title: Functional data validation in PHP
author: Marco Perone @marcoshuttle
patat:
    margins:
        left: 2
        right: 2
    theme:
        syntaxHighlighting:
            keyword: [bold, dullBlue]
...

# Functional data validation in PHP

## Who am I?

Marco Perone

twitter.com/marcoshuttle

github.com/marcosh

marcosh.github.com

## Why data validation?

code with guarantees

current state of validation libraries in PHP

## Why functional?

Good practices

Ability to reason about the code

Compositionality

# What does validation mean?

## Input

                ┌────────────────┐
                │                │
    anything ───│   validation   │
                │                │
                └────────────────┘

Any data (be liberal in what we accept)

## Output

                ┌────────────────┐
                │                │   ┌── validated data
    anything ───│   validation   │───│
                │                │   └── error messages
                └────────────────┘

The validation can either succeed or fail

If the validation succeeds, we can access the validated value

If the validation fails, we can acces the error messages

## Interface

```php
interface Validation
{
    public function validate($data): ValidationResult;
}
```

## ValidationResult

One of two possibilities (success or failure)

No other possibilities

Validated data accessible only if validation succeeds

Error messages available only if validation fails

Force handling of both cases

## ValidationResult

```php
final class ValidationResult
{
    /** @var bool */
    private $isValid;
    
    /** @var mixed */
    private $validContent;
    
    /** @var array */
    private $messages;
    
    private function __construct(bool $isValid, $validContent, array $messages)
    {
        $this->isValid = $isValid;
        $this->validContent = $validContent;
        $this->messages = $messages;
    }
    
    public static function valid($validContent): self
    {
        return new self(true, $validContent, []);
    }
    
    public static function errors(array $messages): self
    {
        return new self(false, null, $messages);
    }
}
```

## ValidationResult

```php
final class ValidationResult
{
    public function process(
        callable $processValid,
        callable $processErrors
    ) {
        if (! $this->isValid) {
            return $processErrors($this->messages);
        }

        return $processValid($this->validContent);
    }
}
```

# Basic examples

## IsString

```php
final class IsString implements Validation
{
    public function validate($data): ValidationResult
    {
        if (! is_string($data)) {
            return ValidationResult::errors(['NO NO-STRING SHALL PASS!']);
        }

        return ValidationResult::valid($data);
    }
}

$isString = new IsString();

$isString->validate('a string'); // ValidationResult::valid('a string')

$isString->validate(42); // ValidationResult::errors(['NO NO-STRING SHALL PASS!'])
```

## NonEmpty

```php
final class NonEmpty implements Validation
{
    public function validate($data): ValidationResult
    {
        if (empty($data)) {
            return ValidationResult::errors(['FEAR THE EMPTYNESS!']);
        }

        return ValidationResult::valid($data);
    }
}
```

# Let's compose!

## Sequence

                ┌────────────────┐                                     ┌────────────────┐                                       
                │                │   ┌── validated data                │                │   ┌── validated data
    anything ───│   validation   │───│                     anything ───│   validation   │───│
                │                │   └── error messages                │                │   └── error messages
                └────────────────┘                                     └────────────────┘

## Sequence

                                                           ┌────────────────┐
                ┌────────────────┐                         │                │   ┌── validated data
                │                │   ┌── validated data ───│   validation   │───│
    anything ───│   validation   │───│                     │                │   └── error messages
                │                │   └── error messages    └────────────────┘
                └────────────────┘

## Sequence

```php
final class Sequence implements Validation
{
    /** @var Validation[] */
    private $validations;

    public function __construct(array $validations)
    {
        $this->validations = $validations;
    }

    public function validate($data): ValidationResult
    {
        return array_reduce(
            $this->validations,
            function (ValidationResult $carry, Validation $validation) {
                return $carry->process(
                    function ($validData) use ($validation) {
                        return $validation->validate($validData);
                    },
                    function () use ($carry) {
                        return $carry;
                    }
                );
            },
            ValidationResult::valid($data)
        );
    }

```

## Sequence

```php
$validator = new Sequence([
    new IsString(),
    new NonEmpty()
]);

$validator->validate(42); // ValidationResult::errors(['NO NO-STRING SHALL PASS!'])

$validator->validate(''); // ValidationResult::errors(['FEAR THE EMPTYNESS!'])

$validator->validate('a string') //ValidationResult::valid('a string')
```

## Other features

Context

Custom error formatters

Translators

Applicative and monadic validation

## Thank you!

twitter.com/marcoshuttle

github.com/marcosh

marcosh.github.com

## Want to know more?

https://github.com/marcosh/php-validation-dsl

http://marcosh.github.io/post/2018/12/03/php-validation-dsl.html

http://marcosh.github.io/post/2019/04/10/applicative-monadic-validation-php.html
