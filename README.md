# Asynchronous JSON Methods Proposal

JavaScript programs are single-threaded and therefore must streadfastly avoid blocking on IO operations. `JSON.parse` and `JSON.stringify` are both blocking operations. For large JSON strings to parse or objects to stringify the blocking time can be very significant.

To avoid blockage of the main thread by `JSON.parse` and `JSON.stringify` we are proposing for asynchronous and non-blocking JSON methods that move the work to a background thread.

## Specification

### `JSON.parseAsync( text [ , reviver ] )`
> Note: this section is mostly copied from [ECMA-262§24.3.1](http://www.ecma-international.org/ecma-262/6.0/#sec-json.parse)

The `parseAsync` function parses a JSON text (a JSON-formatted String) and produces an ECMAScript value. The JSON format is a subset of the syntax for ECMAScript literals, Array Initializers and Object Initializers. After parsing, JSON objects are realized as ECMAScript objects. JSON arrays are realized as ECMAScript Array instances. JSON strings, numbers, booleans, and null are realized as ECMAScript Strings, Numbers, Booleans, and **null**.

The optional *reviver* parameter is a function that takes two parameters, *key* and *value*. It can filter and transform the results. It is called with each of the *key/value* pairs produced by the parse, and its return value is used instead of the original value. If it returns what it received, the structure is not modified. If it returns **undefined** then the property is deleted from the result.

The `parseAsync` function will return a *Promise* object that then will get resolved to result of parsing the JSON.

1. Let JText be ToString(text).
2. ReturnIfAbrupt(JText). >>HELP: Reject?
3. Parse JText interpreted as UTF-16 encoded Unicode points (6.1.4) as a JSON text as specified in ECMA-404 in a background thread (>>HELP added background thread. is that right?). Throw(>>HELP: Reject?) a SyntaxError exception if JText is not a valid JSON text as defined in that specification.
4. Let scriptText be the result of concatenating "(", JText, and ");".
5. Let completion be the result of parsing and evaluating scriptText as if it was the source text of an ECMAScript Script. but using the alternative definition of DoubleStringCharacter provided below. The extended PropertyDefinitionEvaluation semantics defined in B.3.1 must not be used during the evaluation.
6. Let unfiltered be completion.[[value]].
7. Assert: unfiltered will be either a primitive value or an object that is defined by either an ArrayLiteral or an ObjectLiteral.
8. If IsCallable(reviver) is true, then
  * a. Let root be ObjectCreate(%ObjectPrototype%).
  * b. Let rootName be the empty String.
  * c. Let status be CreateDataProperty(root, rootName, unfiltered).
  * d. Assert: status is true.
  * e. Return InternalizeJSONProperty(root, rootName).

  Else
  * a. Return unfiltered. HELP>> Reject?

JSON allows Unicode code units 0x2028 (LINE SEPARATOR) and 0x2029 (PARAGRAPH SEPARATOR) to directly appear in String literals without using an escape sequence. This is enabled by using the following alternative definition of DoubleStringCharacter when parsing scriptText in step 5:

```
DoubleStringCharacter ::
SourceCharacter but not one of " or \ or U+0000 through U+001F
\ EscapeSequence
```
* The SV of DoubleStringCharacter :: SourceCharacter but not one of " or \ or U+0000 through U+001F is the UTF16Encoding (10.1.1) of the code point value of SourceCharacter.

> NOTE: The syntax of a valid JSON text is a subset of the ECMAScript PrimaryExpression syntax. Hence a valid JSON text is also a valid PrimaryExpression. Step 3 above verifies that JText conforms to that subset. When scriptText is parsed and evaluated as a Script the result will be either a String, Number, Boolean, or Null primitive value or an Object defined as if by an ArrayLiteral or ObjectLiteral.


```js
JSON.parseAsync('{"foo": 1}').then(result => console.log(result));
{foo: 1}
```

### `JSON.stringifyAsync(object)`

`JSON.stringifyAsync` will take an object or array and return a promise. The promise will then resolved to a JSON string.

```js
JSON.stringifyAsync({foo: 1}).then(result => console.log(result));
// '{"foo": 1}'
```

### Transpilation / Polyfill

There is no way of transpilating the actual effect of this proposal. But it's very easy to write a polyfill:


```js
JSON.stringifyAsync = JSON.stringifyAsync || function (object) {
  return new Promise((resolve, reject) => {
    try {
      resolve(JSON.stringify(object));
    } catch (error) {
      reject(error);
    }
  });
}

JSON.parseAsync = JSON.parseAsync || function (string) {
  return new Promise((resolve, reject) => {
    try {
      resolve(JSON.parse(string));
    } catch (error) {
      reject(error);
    }
  });
}
```
