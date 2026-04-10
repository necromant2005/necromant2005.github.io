---
title: Why I Hate Tailwind and Love Bootstrap
permalink: /dev/why-i-hate-tailwind-and-love-bootstrap/
---

# Why I Hate Tailwind and Love Bootstrap

Frontend debates often reduce to style preferences, but this one is about how complexity behaves over time.
Tailwind CSS looks like a standard choice in modern stacks (Vite, Webpack, React, Vue, Next.js), especially in startup and SaaS environments, not because it is inherently better, but because it aligns with how those systems are built.
On the other side, Bootstrap feels like a product of the older HTML-centric era, where pages were assembled rather than composed as systems.

## The illusion of control

Tailwind CSS gives a strong sense of control because every visual decision is explicit and local, but that control becomes deceptive as the system grows.

```html
<div class="flex flex-col md:flex-row items-center justify-between p-4 gap-2 md:gap-4 bg-white border rounded-lg shadow-sm">
  <button class="px-3 py-1 text-sm md:text-base bg-blue-500 hover:bg-blue-600 text-white rounded">
    Buy
  </button>
</div>
```

At small scale, this works well, but structurally it turns markup into a distributed configuration system where layout, spacing, and behavior are fragmented across many tokens.

As a result:

- Small changes require editing multiple places
- Dependencies are implicit rather than structured
- Layout rules spread across components
- Side effects become difficult to predict

Nothing fails immediately, but the system becomes harder to reason about.

---

## The value of constraints

Bootstrap takes the opposite approach by limiting flexibility and embedding decisions into predefined components.

```html
<div class="card p-3">
  <button class="btn btn-primary">
    Buy
  </button>
</div>
```

Here, spacing, sizing, responsiveness, and interaction are already defined. Instead of assembling styles manually, you rely on a system that encodes consistent behavior.

This leads to:

- Clear component boundaries
- Predictable rendering across devices
- Fewer layout regressions
- Reduced debugging surface

---

## Core difference

<div class="table-responsive">
  <table class="table table-striped table-bordered align-middle">
    <thead class="table-light">
      <tr>
        <th scope="col">Aspect</th>
        <th scope="col">Tailwind</th>
        <th scope="col">Bootstrap</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <th scope="row">Philosophy</th>
        <td>Compose everything</td>
        <td>Use predefined system</td>
      </tr>
      <tr>
        <th scope="row">Control</th>
        <td>High</td>
        <td>Limited</td>
      </tr>
      <tr>
        <th scope="row">Predictability</th>
        <td>Decreases with scale</td>
        <td>Remains stable</td>
      </tr>
      <tr>
        <th scope="row">Maintenance</th>
        <td>Hidden cost</td>
        <td>Explicit cost</td>
      </tr>
      <tr>
        <th scope="row">Failure mode</th>
        <td>Layout drift</td>
        <td>Visual rigidity</td>
      </tr>
    </tbody>
  </table>
</div>

---

## Where complexity shows up

Tailwind makes development fast at the beginning by pushing decisions into markup, but that complexity accumulates over time and becomes harder to manage.

Bootstrap introduces constraints upfront, which slows initial flexibility but keeps the system stable as it grows.

---

## The actual distinction

Tailwind optimizes for writing UI quickly.
Bootstrap optimizes for maintaining UI reliably.
