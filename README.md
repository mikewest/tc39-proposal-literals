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

The platform could provide developers with this kind of type check in a number of ways. Two that
seem viable are:

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

## FAQ

*   __Can't folks just do this themselves with Closure, or other build-time checks?__

    Yes, and in fact Google _is_ doing this today internally. Folks involved in that build process
    would like to make it more robust by layering client-side checks on top of the build-time
    analysis they're already doing. The `Trusted Types` proposal mentioned above would also benefit
    from a `fromLiteral` construction mechanism, as the anecdata from Google's codebase shows that
    that mechanism is both safe and widely usable.

## Prior Art

*   [`goog.string.Const`](https://google.github.io/closure-library/api/goog.string.Const.html) is
    supported in the Closure compiler.
*   GWT provides [`SafeHtml.fromSafeConstant`](http://www.gwtproject.org/javadoc/latest/com/google/gwt/safehtml/shared/SafeHtmlUtils.html#fromSafeConstant-java.lang.String-), also via compile-time checks.
*   https://github.com/google/safe-html-types/blob/master/doc/safehtml-types.md shows usage of similar
    concepts in C++ (`TrustedResourceUrl::FromConstant`).
