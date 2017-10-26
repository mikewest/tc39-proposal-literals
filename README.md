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
that generates typed objects. The [Trusted Types](https://github.com/mikewest/trusted-types) proposal aims
to do just that.

For the most part, this mechanism is quite possible to implement entirely in DOM and WebIDL, without
touching the underlying language at all. However, one distinction that security reviewers inside Google
have found critical isn't currently possible to replicate on the web without some language-level hooks.

The Closure compiler can distinguish between strings that are embedded into a program as literals, and
those that are the result of some operation (method call, property getter, etc). It enforces constraints
on [`goog.string.Const`](https://google.github.io/closure-library/api/goog.string.Const.html) which ensure
that such objects can only be constructed from literals, which a) is apparently quite common in production
code at Google, and b) is provably safe (attacker cannot control the value of the literal without injecting
into the script creating such literal).

## Proposal

It would be lovely if the language would provide a mechanism that WebIDL could hook into in order to
distinguish string literals when performing type checks. That is, today, we can produce something like
the following snippet:

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

## Prior Art

*   [`goog.string.Const`](https://google.github.io/closure-library/api/goog.string.Const.html) is
    supported in the Closure compiler.
*   GWT provides [`SafeHtml.fromSafeConstant`](http://www.gwtproject.org/javadoc/latest/com/google/gwt/safehtml/shared/SafeHtmlUtils.html#fromSafeConstant-java.lang.String-), also via compile-time checks.
*   https://github.com/google/safe-html-types/blob/master/doc/safehtml-types.md shows usage of similar
    concepts in C++ (`TrustedResourceUrl::FromConstant`).
