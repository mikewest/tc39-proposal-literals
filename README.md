# Literals in script

## Motivation

HTML's DOM offers a number of mechanisms to turn arbitrary strings into markup (`.innerHTML = ...`) or code
(`scriptEl.innerText = ...`, `el.onclick = ...`, etc). Each of these mechanisms can serve as an XSS sink,
giving an attacker the ability to feed code into a context that wasn't expecting it, leading to a class of
DOM-based XSS attacks that we'd very much like to avoid.

One way of addressing this issue that's [worked well in companies like
Google](https://research.google.com/pubs/pub42934.html) is to move away from the string-based APIs, towards
a more strongly-typed interface that could enforce some degree of validation and sanitization at the point
at which a string enters into the application. If developers can lock themselves into such a system, then
they can reduce the necessity to deeply audit each usage of a known XSS sink, and instead focus on the code
that generates typed objects like `SafeHtml` or `SafeUrl`. The
[Trusted Types](https://github.com/mikewest/trusted-types) proposal aims to do just that.

For the most part, this mechanism is quite possible to implement entirely in DOM and WebIDL, without
touching the underlying language at all. However, one distinction that security reviewers inside Google
have found critical isn't currently possible to replicate on the web without some language-level hooks.

The Closure compiler can distinguish between strings that are embedded into a program as literals, and
those that are the result of some operation (method call, property getter, etc). It enforces constraints
on [`goog.string.Const`](https://google.github.io/closure-library/api/goog.string.Const.html) which ensure
that such objects can only be constructed from literals, which a) is apparently quite common in production
code at Google, and b) provably safe (attacker cannot control the value of the literal without injecting
directly into the script creating such literal).

That is, developers might create a `SafeUrl` object by first creating a `goog.string.Const`, and then using
it in a factory method, like:

```js
const url = goog.string.Const.from("https://safe.test/totally/safe/url");
return SafeUrl.fromConstant(url);
```

It would be lovely if we could make this assertion in the platform, rather than relying entirely on
build-time checks. Client-side assertions could increase the robustness of the checks, providing
defense in depth, and a safety net for the cases where code sneaks its way through the release
process without being compiled.

## Proposals

I honestly don't have enough context with the innards of the language to make defensible proposals.
Instead, I have the use cases above, and vague sketches of what I'd like to see from the platform side.
I'd love to get feedback from folks who actually know things about the language with regard to the
details of what the language might be willing and able to provide.

With that in mind: wild speculation about what might or might not be reasonable follows!

### A Literal String Type

One way of providing this capability would be to expose a new string type corresponding to literal
strings, which WebIDL could build on top of in order to distinguish literals when performing type
checks. That is, today, we can produce something like the following WebIDL snippet:

```
interface TrustedHTML {
  static TrustedHTML escape(DOMString html);
};
```

This can be called as `TrustedHTML.escape("Literal string!")` and as
`TrustedHTML.escape(formField.value)`, escaping the string as necessary.

It would be ideal if we could add something like:

```
interface TrustedHTML {
  static TrustedHTML createFromLiteral(LiteralString html);
};
```

This could be called as `TrustedHTML.createFromLiteral("Literal string!")`, but calling
`TrustedHTML.createFromLiteral(formField.value)` would throw a `TypeError`.

Also ideally, we'd allow literals to be combined with other literals (which is commonplace
in codebases with agreed-upon line-length limits). That is, we'd allow the following:

```js
TrustedHTML.createFromLiteral("A marginally longer literal string that seems to keep going " +
                              "and going and going and going. Wow, what a long string.");
```

Equally ideally, we'd keep track of a string's literalness. That is, we'd allow the following:

```js
let a = "Literal string!";
TrustedHTML.createFromLiteral(a);

let b = "Another literal!";
TrustedHTML.createFromLiteral(a + b);

let c = a + b;
TrustedHTML.createFromLiteral(c);
```

### Strict Tag Functions

One alternative which comes to mind would be to build a similar system on top of tagged template
strings. That is, you could imagine something like:

```js
function trustedUrlizer(templateString) {
  return TrustedUrl.unsafelyCreate(templateString);
}

return trustedUrlizer`https://safe.test/totally/safe/url`;
```

This would be pretty great iff we also had a mechanism to ensure that the tag function would _only_
accept template strings. That is, `trustedUrlizer(formField.value)` would fail, perhaps throwing a
`TypeError`.

Daniel Ehrenberg refined this a bit in
[mikewest/tc39-proposal-literals#2](https://github.com/mikewest/tc39-proposal-literals/issues/2),
suggesting:

> Here's the API surface I'm imagining: the first argument to a template tag would have an additional property, `literal`, which indicates whether the string provided as an argument to the template was a literal. We could make this unforgeable by representing this as an internal slot, with `literal` as an own getter, so you could do something like this to prevent an attacker from calling your template with any old object that has a `literal: true` property:
>
> ```js
> // Un-monkey-patchable way to get the getter
> let getLiteral = Object.getOwnPropertyDescriptor((_ => _)``, "literal").get; 
> 
> function literalString(strings, ...keys) {
>   if (!getLiteral.call(strings)) throw new Error();
>   return String.raw(strings, ...keys);
> }
> ```
> 
> This `literalString` template tag is like `String.raw` but would throw an exception if you don't pass a literal. It outputs a normal string, with no trace any more that it was literal. This two-liner could be the recommended way to set off a call to something that requires a literal string. It's not possible to compromise the string literal contents because the `strings` object (and its inner `raw` object) are frozen.

## FAQ

*   __Can't folks just do this themselves with Closure, or other build-time checks?__

    Yes, and in fact Google _is_ doing this today internally. Folks involved in that build process
    would like to make it more robust by layering client-side checks on top of the build-time
    analysis they're already doing. The `Trusted Types` proposal mentioned above would also benefit
    from a `fromLiteral` construction mechanism, as the anecdata from Google's codebase shows that
    that mechanism is both safe and widely usable.

*   __To what extent would we need to track literalness for the first proposal?__

    An excellent question! I'm flexible!

    One thing we _don't_ need to track would be something like using a literal as an object key.
    That is, I'm perfectly happy treating `{ "a": "value" }` and `{ a: "value" }`, and
    `obj["a"] = "value"` as having a value whose key is the same.

## Prior Art

*   [`goog.string.Const`](https://google.github.io/closure-library/api/goog.string.Const.html) is
    supported in the Closure compiler.
*   GWT provides [`SafeHtml.fromSafeConstant`](http://www.gwtproject.org/javadoc/latest/com/google/gwt/safehtml/shared/SafeHtmlUtils.html#fromSafeConstant-java.lang.String-), also via compile-time checks.
*   https://github.com/google/safe-html-types/blob/master/doc/safehtml-types.md shows usage of similar
    concepts in C++ (`TrustedResourceUrl::FromConstant`).
