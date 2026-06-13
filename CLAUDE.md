# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Sonder is a static portfolio website for an indie game developer. Currently a single-page site built with vanilla HTML, CSS, and JavaScript — no frameworks, no build tools, no dependencies. The site is served directly from `website/index.html`.

## Repository structure

```
website/index.html   — The entire site: CSS design system, HTML content, GLSL shaders, and JS application code
```

## Development

There is no build step, no package manager, and no test suite. To preview the site locally, open `website/index.html` in a browser, or run a static file server from the `website/` directory:

```bash
cd website && python3 -m http.server 8080
# then visit http://localhost:8080
```

## Architecture

### CSS design system (inline `<style>`)

Custom properties define a neon-on-dark theme with glassmorphism panels (`:root` block, line ~17). Key tokens: `--neon-cyan`, `--neon-magenta`, `--neon-purple`, `--neon-green`, `--glass-bg`, `--glass-blur`. The `.glass-panel` class applies the frosted-glass effect with backdrop-filter and fallback for browsers that don't support it. Sections animate in via `IntersectionObserver` (`.section` opacity/transform transition on `.visible`).

### WebGL2 fluid simulation (`FluidSimulation` class)

The background is a real-time 2D Navier-Stokes fluid solver running on a 256×256 (128×128 on mobile) grid via WebGL2. The simulation pipeline per frame:

1. **Splat** dye and velocity from mouse/touch/idle input (additive blending)
2. **Vorticity confinement** — preserves swirling detail
3. **Advection** — semi-Lagrangian transport for velocity and dye
4. **Divergence** computation
5. **Pressure solve** — Jacobi iteration (20 iterations desktop, 10 mobile)
6. **Gradient subtraction** — enforces incompressibility
7. **Render** — composite dye with bloom, vignette, film grain, chromatic aberration, and Reinhard tone mapping

Shader programs are defined as `<script type="x-shader/x-vertex">` and `<script type="x-shader/x-fragment">` elements and compiled at runtime via `_compileShader` / `_createProgram`. GLSL is ES 3.00 (`#version 300 es`).

The simulation uses a double-buffering scheme: velocity, dye, and pressure each have two textures (`velTex[0/1]`, `dyeTex[0/1]`, `pressureTex[0/1]`). The `readIdx` object tracks which buffer is the current read source, and `_swap()` flips them between passes.

### App controller (`App` class)

- Initializes `FluidSimulation` with parameters tuned for mobile vs desktop (`_isMobile()` — width < 768px or touch + width < 1024px)
- Falls back to a CSS-animated radial gradient (`.webgl-fallback`) when WebGL2 is unavailable or context is lost
- Handles WebGL context loss/restoration — recreates the entire simulation on restore
- Pauses the animation loop when the tab is hidden (`visibilitychange`)
- Manages idle auto-splatting (every 8 seconds) to keep the background alive when the user isn't interacting
- Smooth-scrolls anchor links with nav offset compensation
