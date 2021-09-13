# Regular Expression Language (RegXL)

## Introduction

The standard Regular Expression syntax is a relatively simple language, but it has the problem of becoming illegible and hard to maintain very quickly. For example, a simplified IPv4 Address expression could be represented as such:

```js
/(25[0-5]|2[0-4]\d|1?\d?\d)(\.(25[0-5]|2[0-4]\d|1?\d?\d)){3}/
```

How long would it take a maintainer to review the expression and troubleshoot it for possible errors? The expression is not self-documenting and most wouldn't take the time to analyze the expression to validate its correctness. Wouldn't it be better if we could represent the same expression in a simpler way?

```abnf
4x( number(0 to 255) separator '.' )
```

But how could you turn that into something useable within Javascript? Here is an example of searching an input string for all IPv4 addresses:

```js
const regex = regxl`
	4x( number(0 to 255) separator '.' )
`

let matches = input.matchAll(regex)
```

## Syntax

The expression syntax look similar to BNF and has similar properties. White space is generally ignored so you can structure the expression to make it easier to understand.

## Reusability? Extensibility?

You can add custom tokens to the query language by defining its behaviour in a function and passing it into your RegXL query to be parsed. Below is an example custom token that searches for simple HTML elements without any attributes or nested elements.

```js
const HtmlElementXL = (content) => ({
	statement: [
		`'<' #tagName(letter letterNumber*) '>'`,
		content,
		`'</' @tagName '>'`
	]
})

const regexp = regxl({htmlElement: HtmlElementXL})`
	htmlElement( 'My Page Title' )
`

const input = '<title>My Page Title</title>'

console.log(regexp.test(input)); // => true
```

There are further details below on how you can use this powerful extension capability.


## What about performance?

RegXL does add some small overhead for parsing, but it does [cache](#caching) the results by default so if the same query where to be accessed again it wouldn't have to parse it a second time. The speed of the generated expression is identical to one that would be hand written so query parsing is the only overhead. Alternatively, you can add a step into your build process to generate the resultant RegExp ahead of time.

If you are concerned about [parsing speed](https://medium.com/textmaster-engineering/performance-of-regular-expressions-81371f569698) in your application then Regular Expressions may be the wrong choice to begin with. Optimizing your regular expressions for peak speed can be an artform in itself and may not be worth the effort compared to other string parsing techniques.

> # TODO: Add a way to tie this into common build processes (webpack, etc)

## Differences from standard Regular Expressions

The exisitng syntax does not promote easy use of non-latin languages. For instance, `\w` and `\d` only match against latin letters and arabic numerals. Using the Unicode versions involves more steps such as looking up the correct property names and remembering to avoid the easier legacy shortcuts.

The standard RegXL behaviour is to match Unicode characters when possible when statements such as `letter` and `digit` are used. You can revert back to legacy behaviour (not recommended) by specifying `with binary` to remove all Unicode matching.


# Assertions

| RegXL		| RegExp	| Description
| ---		| --- 		| ---
| `start`	| `^` 		| Start of the input string.
| `end`		| `$`		| End of the input string.
| `startLine`| `^` with `m` modifier| Start of a line.
| `endLine`	| `$` with `m` modifier	| End of a line.
| `alphaNumericBoundary`| `\b`		| Boundary between a word and non-word character.
| `not alphaNumericBoundary`| `\B`	| Not a boundary between character types.
| `letterBoundary` 
| `followedBy` | `(?= )`| Lookahead.
| `not followedBy` | `(?! )`| Negative Lookahead.
| `precededBy` | `(?<= )`| Lookbehind.
| `not precededBy` | `(?= )`| Negative Lookbehind.



# Character Classes

These indicate specific characters that fall into a particular category. Most of these can be inverted by adding a `not` in front of them.

## Generic

| RegXL		| Supports `not`	| RegExp	| Description	|
| ---		| --- 				| ---		| --- 			|
| `any`		| ✓		| `.`		| Any character except line terminators.|
| `anything`| ✖		| `[^]`		| Any character.			|

## Numbers
> **Note** These do not support international locales but are intended to be compatible with the [syntax used in Javascript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number#literal_syntax).

| RegXL		| RegExp	| RegExp `with binary`		| Description
| ---		| ---		| ---						| ---
| `digit`	| `\d`		| Digit 0-9					|
| `numeric` | `\p(Numeric_Value}`
| `integer` | `[-+]?\d+`| Positive or negative whole number. |
| `+integer`| `[+]?\d+`	| Positive integer.			|
| `-integer`| `-\d+`	| Negative integer.			|
| `decimal` | `[-+]?\d+(\.\d+)?` | Decimal number.		|
| `alphaNumeric`| `\p{Alphabetic}\|\p{Numeric_Value}`| `[A-Za-z0-9]`


## Letters
| RegXL		| Supports `not`| RegExp			| RegExp `with binary`
| ---		| ---			| ---				| ---
| `ascii`	| ✓				| `\p{ASCII}`		| `[\x00-\x7F]`
| `letter`	| ✓				| `\p(Alphabetic}`	| `[A-Za-z]`
| `upperLetter`	| ✓			| `\p(Uppercase}`	| `[A-Z]`
| `lowerLetter`	| ✓			| `\p(Lowercase}`	| `[a-z]`
| `emoji`	| ✓				| `\p(Emoji}`		| `Error`


## Spacing
| RegXL		| Supports `not`	| RegExp	| RegExp `with binary`
| ---		| --- 				| ---		| ---
| `whitespace` | ✓	| `\p(White_Space)`		| `\s`
| `space`	| ✓		| `[ ]`					|
| `tab`		| ✓		| `\t`
| `tabSpace`| ✓		| `[\t ]`
| `newline` | ✓		| `\r?\n\|\r\|\u{2028}\|\u{2029}`| `\r?\n\|\r`
| `null`	| ✓		| `\u{0000}`			| `\x00`


# Groups and Ranges

| RegXL		| RegExp		| Description	| Examples
| ---		| ---			| --- 			| ---
| `or`		| `\|`			| Matches the left _or_ the right expression.
| `oneOf( )`| `[ ]` or `\|`	| Matches exactly one of the enclosed expressions. | `oneOf( digit 'A'to'Z' )`
| character ` to ` character| `[ - ]` | | `'A' to 'Z'` or `\x00 to \n`
| `( )`		| `(?: )`		| Non-capturing group.
| `group( )`| `( )`			| Capturing group.
| `#name( )`| `(?<name> )`	| Named capture group.
| `@n`		| `\n`			| Numerical back reference.
| `@name`	| `\k<name>`	| Named back reference.

## Ranges Alternatives

Sometimes there are multiple ways to represent a range. Below are some common examples of representing the same expression in different ways.

| Standard RegXL| Example | Alternative | Alternative Example |
| --- 			| --- | --- | ---
| `oneOf( )`	| `oneOf([ab] 'm' 'n' 'x' to 'z')`	| `[ ]`		| `[ab] or [mn] or [x-z]`
| `not oneOf( )`	| `not oneOf([ab] 'm' 'n' 'x' to 'z')`	| `[^ ]`		| `[^abmnx-z]` or `not [abmnx-z]`
| character `to` character | `'a' to 'z'`| `[ - ]`	| `[a-z]`
| `not` character `to` character | `not 'a' to 'z'`| `[^ - ]`	| `[^a-z]`


# Quantifiers

Greedy quantifiers will continue matching occurances of itself as much as possible. The non-greedy variances will only match the minimum number of times that will satisfy the expression.

For example, for the expression `3+(★)` and the input `★★★★★` the greedy version would match all five, while the non-greedy would only match the minimum of three.

| RegXL		| RegExp	| Greedy| Minimum	| Maximum	| Description	|
| ---		| ---		| ---	| ---		| ---		| --- 			|
| `optional`| `?`		| ✓		| 0			| 1			| Match the following expression **if possible**.
| `optional asMany`	| `*`		| ✓		| 0			| No Limit	| Match the following expression **as many times as possible**, or not at all.
| `asMany`	| `+`		| ✓		| 1			| No Limit	| Match the following expression **as many times as possible**, but at least once.
| `maybe`	| `??`		| 		| 0			| 1			| Match the following expression **if needed**.	
| `optional many`	| `*?`		| 		| 0			| No Limit	| Match the following expression **as few times as needed**.
| `many`	| `+?`		| 		| 1			| No Limit	| Match the following expression **as few times as needed**, but at least once.
| _n_`x`		| `{n}`		|		| _n_		| _n_		| Match the following expression the specified number of times exactly.
| _n_`+`		| `{n,}`	| ✓		| _n_		| No Limit	| Match the following expression **as much as possible**, but at least _n_ times.
| _n_`-`_m_		| `{n,m}`	| ✓		| _n_		| _m_		| Match the following expression **as much as possible** but at least _n_ times and no more than _m_ times. 
| `fewest `_n_`+`		| `{n,}?`	|		| _n_		| No Limit	| Match the following expression **as few times as needed**, but at least _n_ times.
| `fewest `_n_`-`_m_ | `{n,m}?` |		| _n_		| _m_		| Match the following expression **as few times as needed** but at least _n_ and no more than _m_ times.

> # TODO: Possesive? .++  .*+ < don't give back once matched. greedy, lazy, possesive.

## Quantifier Alternatives

Sometimes there are multiple ways to represent a quantifier. Below are some common examples of representing the same expression in different ways.

> **Note:** Care must be taken when mixing styles such that a misplaced `?` doesn't mistakenly become an unintended `optional` or `non-greedy` specifier.


| Standard RegXL| Alternative #1 | Alternative #2 |
| --- 			| --- | ---
| `asMany`		| `1+( )`		| `( )+`
| `many`		| `fewest 1+( )`| `( )+?`
| `optional asMany` | `0+( )`	| `( )*`
| `optional many`	| `fewest 0+( )` | `( )*?`
| `optional`		| `( )?`
| `maybe`			| `( )??`
| `nx`				| `( ){n}`	| `n*`
| `n+`				| `( ){n,}`
| `fewest n+`		| `( ){n,}?`
| `n-m`				| `( ){n,m}`
| `fewest n-m`		| `( ){n,m}?`



# Regular Expression Modifiers

There are some default regular expression modifiers enabled in the resultant RegExp. The goal with the default modifiers is to support the majority of use cases.

| Modifier	| Definition |
| ---		| ---
| `g`		| Global match.
| `u`		| Unicode matching.

Additional modifiers can be enabled, or default modifiers can be disabled, with the following keywords. These are added by placing the `with` keyword at the end of the definition followed by one or more of the following. For example, for `binary` and `indices` you would append `with ( binary indices )` to your  RegXL definition.

| RegXL		| Modifier	| Definition
| ---		| ---		| ---
| ignoreCase | `i`		| Ignores the case of all letter characters.
| binary	| not `u`	| Removes the `u` unicode modifier.
| indices	| `d`		| Generates indices for group matches.

There are a few modifiers that are not relevant when using RegXL since their behaviour is implemented in different ways.

| RegExp Modifier	| RegXL Alternative |
| ---				| ---
| `m`				| `start` and `end` always match the entire input string while `startLine` and `endLine` always match the line.
| `s`				| `any` and `anything` match without and with line terminators respectively.
| `y`				| Limited use case that isn't handled.
| not `g`			| Limited use case that isn't handled.
