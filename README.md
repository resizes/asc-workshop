# asc-workshop

Materials for **Asturias Software Crafters** workshop sessions. The main deck is an **LLMOps platform** session: running and governing AI systems in production, not just demos.

## Slides (Slidev)

The workshop deck lives at the **repository root** (`slides.md`, `public/`, `layouts/`, …). Built with [Slidev](https://sli.dev/).

```bash
npm install
npm run dev
```

That starts the dev server and opens the browser.

```bash
npm run build
```

outputs a static site to **`dist/`**.

```bash
npm run export
```

produces PDF (see Slidev [export docs](https://sli.dev/guide/exporting.html)).

## Workshop manifests

- **`kind/`** — local **kind** + **vLLM** manifests (Part 1).
- **`eks/`** — **EKS** cluster config, Helm values, and **LiteLLM** / **Ollama** manifests (Part 2).

