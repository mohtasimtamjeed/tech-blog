---
author: Mohtasim Tamjeed
pubDatetime: 2026-11-15T23:59:01Z
title: Blog, Explain Yourself!
postSlug: blog-explain-yourself
featured: true
draft: false
tags:
  - meta
  - astro
  - cicd
  - cloudflare
  - systems
description: A technical manifest of the architecture, stack decisions, math engines, and CI/CD delivery pipeline powering this blog.
---

Every engineering project should justify its existence and its architecture.

This site is not running on WordPress, Medium, or Substack. It is a compiled, edge-delivered static artifact designed to solve a specific problem: writing about physics, mathematics, and systems programming without wrestling with clunky WYSIWYG editors, broken equation parsers, or hosting bills.

Here is the technical specification of the entire system.

---

## 1. The Architecture

The system treats prose like software: version-controlled, validated against static schemas, built in an isolated CI runner, and shipped as immutable static artifacts.

```
+-------------------------------------------------------------+
|                     Local Machine (VS Code)                 |
|             Markdown + KaTeX + Co-located Assets            |
+-------------------------------------------------------------+
                              |
                              | git push origin main
                              v
+-------------------------------------------------------------+
|                      GitHub Repository                      |
|                  Source code, themes, posts                 |
+-------------------------------------------------------------+
                              |
                              | Webhook: push / PR
                              v
+-------------------------------------------------------------+
|              GitHub Actions CI/CD Pipeline                  |
|                                                             |
|   [STAGE 1: Automated Quality Gate & Testing]               |
|   ├── npm ci (clean dependency hydration)                   |
|   ├── ESLint & Prettier (code style & syntax linting)       |
|   ├── astro check (Zod frontmatter validation & types)      |
|   └── build test (rehype-katex & AST compilation test)      |
|                                                             |
|                    ── ALL CHECKS PASS ──                    |
|                              │                              |
|   [STAGE 2: Production Deploy (main branch only)]           |
|   └── Wrangler Action pushes ./dist artifact to Cloudflare  |
+-------------------------------------------------------------+
                              |
                              | Direct static artifact upload
                              v
+-------------------------------------------------------------+
|                 Cloudflare Pages Edge CDN (PoPs)            |
|                 Instant, zero-latency static HTML           |
+-------------------------------------------------------------+
```

---

## 2. Why This Stack?

### Why Astro over Hugo or Substack?

- **Build-Time Math:** Substack treats equations as an afterthought. Hugo executes client-side KaTeX scripts that cause a visible layout flash as the browser parses text. Astro compiles equations to pure HTML/MathML during build time. The reader's browser downloads zero client-side math libraries.

- **Component Extensibility (MDX):** If I want to embed an interactive canvas, a dynamic simulation slider, or WebAssembly (Pyodide) in a future post, Astro lets me drop a UI component straight into Markdown.

### Why Markdown (`.md`)?

- **Portability:** Markdown is plain text. If Cloudflare disappears tomorrow, or if I switch static site generators in five years, the raw content remains mine—unlocked from proprietary databases or proprietary CMS schemas.

### Why GitHub Actions over Native Cloudflare Builds?

- **Strict Quality Gates:** Instead of blindly deploying whatever is pushed, the CI runner validates post frontmatter schemas via `astro check` (catching missing tags, incorrect datetime formats, or invalid types) before anything reaches production.
- **Decoupled Infrastructure:** The build environment is decoupled from the hosting layer. Cloudflare only receives dumb, compiled static artifacts (`.html`, `.css`, `.js`). If hosting ever shifts to AWS S3/CloudFront or a self-hosted Nginx container, the build logic and pipeline remain completely unchanged.
- **Automated CI Audits:** Provides a clean integration surface to plug in automated link verification, markdown linting, or asset compression steps down the line.

---

## 3. Supported Primitives

### Mathematical Proofs ($\LaTeX$ via KaTeX)

Inline equations sit flush with prose: $\mathcal{H} = \mathbf{p} \cdot \dot{\mathbf{q}} - \mathcal{L}$.

Complex multi-line derivations compile cleanly without layout shift:

$$\frac{\partial \rho}{\partial t} + \nabla \cdot (\rho \mathbf{u}) = 0$$

$$\rho \left( \frac{\partial \mathbf{u}}{\partial t} + \mathbf{u} \cdot \nabla \mathbf{u} \right) = -\nabla p + \mu \nabla^2 \mathbf{u} + \mathbf{f}$$

### Syntax-Highlighted Code Blocks

Code blocks run through Shiki with active theme highlighting:

```python
import numpy as np

def symplectic_euler(q0, p0, dt, steps, grad_v):
    """First-order symplectic integrator preserving phase volume."""
    q, p = np.zeros(steps), np.zeros(steps)
    q[0], p[0] = q0, p0
    for i in range(steps - 1):
        p[i + 1] = p[i] - dt * grad_v(q[i])
        q[i + 1] = q[i] + dt * p[i + 1]
    return q, p
```

### Executable Interactive Scripting & CI Verification

Because Astro natively supports client-side `<script>` tags and `.mdx` components, posts aren't restricted to static code blocks. I can embed direct, executable JavaScript snippets to drive interactive physics toys, parameter sliders, or client-side visualizers:

```html
<div id="integrator-demo" class="border-skin-line rounded border p-4">
  <button
    id="step-btn"
    class="bg-skin-accent text-skin-inverted rounded px-3 py-1"
  >
    Step Integrator
  </button>
  <span id="output" class="ml-3 font-mono">q: 1.000, p: 0.000</span>
</div>

<script>
  let q = 1.0,
    p = 0.0,
    dt = 0.05;
  document.getElementById("step-btn")?.addEventListener("click", () => {
    p -= dt * Math.sin(q);
    q += dt * p;
    const out = document.getElementById("output");
    if (out) out.textContent = `q: ${q.toFixed(3)}, p: ${p.toFixed(3)}`;
  });
</script>
```

<div class="my-6 rounded-lg border border-skin-line p-4 bg-skin-card">
  <div class="flex flex-wrap items-center gap-4">
    <button
      id="live-step-btn"
      type="button"
      class="rounded bg-skin-accent px-4 py-2 font-mono text-sm font-semibold text-skin-inverted transition hover:opacity-90 active:scale-95"
    >
      Step Integrator
    </button>
    <button
      id="live-reset-btn"
      type="button"
      class="rounded border border-skin-line px-3 py-2 font-mono text-sm opacity-80 hover:opacity-100"
    >
      Reset
    </button>
    <span id="live-output" class="font-mono text-sm text-skin-accent">
      q: 1.000, p: 0.000 | step: 0
    </span>
  </div>
  <div class="mt-3 text-xs opacity-70 font-mono">
    Euler-Cromer / semi-implicit step over dV/dq = sin(q)
  </div>
</div>

<script>
  (() => {
    let q = 1.0;
    let p = 0.0;
    let step = 0;
    const dt = 0.1;

    const out = document.getElementById("live-output");
    const stepBtn = document.getElementById("live-step-btn");
    const resetBtn = document.getElementById("live-reset-btn");

    stepBtn?.addEventListener("click", () => {
      // Symplectic kick-drift step:
      p -= dt * Math.sin(q);
      q += dt * p;
      step++;
      if (out) {
        out.textContent = `q: ${q.toFixed(3)}, p: ${p.toFixed(3)} | step: ${step}`;
      }
    });

    resetBtn?.addEventListener("click", () => {
      q = 1.0;
      p = 0.0;
      step = 0;
      if (out) {
        out.textContent = `q: ${q.toFixed(3)}, p: ${p.toFixed(3)} | step: 0`;
      }
    });
  })();
</script>

Running embedded logic inside articles usually introduces the risk of client runtime crashes or unhandled DOM errors slipping into production.

Because the deployment runs through GitHub Actions before reaching Cloudflare, every embedded script, AST node, and TypeScript type is evaluated by astro check and ESLint in the runner. If an embedded component has broken types, an unclosed tag, or an invalid dependency import, the CI pipeline halts and rejects the deployment before broken code ever reaches the CDN.

### Media & Reaction Assets

Media files can be dropped directly alongside posts and referenced locally via standard Markdown tags:

```markdown
![System Status](@/assets/images/dumpster_fire.gif)
```

![System Status](@/assets/images/dumpster_fire.gif)
---

## 4. Ground Rules

1. **Depth over fluff:** No generic AI-generated filler.
2. **First principles:** Complex systems explained via clear derivations, working code, or physical analogies.
3. **Rough drafts out in public:** Learning, building, and breaking things out loud.
