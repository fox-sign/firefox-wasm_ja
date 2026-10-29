# Web deployment (Firefox WASM + Japanese fonts)

This project is a WebAssembly build of Gecko/Firefox. It is not a static HTML-only app: the chrome demo needs the compiled engine assets and a WebSocket WISP proxy for browsing external sites.

## 1. Build on Linux

Use Linux (native Linux or a Linux VM/WSL2). Follow the root README prerequisites first: emsdk 6.0.1, Rust with `wasm32-unknown-emscripten` and rust-src, Python 3, Node.js and pnpm.

From the repository root:

```bash
make chrome-demo
```

This builds the engine, stages `demo/chrome/public/chrome-assets.tar.zst`, and starts the Vite development server. The chrome-assets step downloads Noto Sans JP from Google Fonts and puts it in both `fonts/` and `browser/fonts/` in the archive. The Firefox font-list code scans the application `fonts/` directory, and `chrome-fs.ts` fails fast if the Japanese font is missing.

Open the local URL printed by Vite. Keep the server running while using the browser. Cross-origin isolation headers (COOP/COEP) and the WISP WebSocket endpoint are required.

## 2. Build a production site

In a separate terminal, from the repository root:

```bash
pnpm install
make libxul
make chrome-assets
pnpm --filter chrome-demo build
```

The static site files are generated in `demo/chrome/dist/`. Upload that directory to your web host, but **the static files alone are not enough**: configure a WebSocket-capable WISP proxy at the same origin under `/wisp/`. The demo's Vite configuration runs that proxy only for development/preview; a production deployment must provide an equivalent WISP service and route `wss://YOUR_DOMAIN/wisp/` to it.

Your web server must send these headers for the application pages and WASM responses:

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Serve `.wasm` as `application/wasm`, and ensure `.wasm.zst`, `.tar.zst`, and the JSON manifest are served as binary / JSON files without HTML fallback. Use HTTPS in production. If hosting under a subpath, preserve the relative files in `dist/` and route the WebSocket endpoint under that same origin/path correctly.

## 3. Hosting notes

- GitHub Pages can host the static `dist/` files, but it cannot run the required WISP WebSocket proxy. It is therefore **not a complete deployment by itself** for browsing arbitrary sites.
- Use a VPS or a hosting platform that supports a persistent Node.js/WebSocket service and custom response headers, or host the static files and WISP proxy behind a reverse proxy on the same origin.
- The first build downloads a large Emscripten toolchain and compiles Gecko; allow substantial disk space, RAM, and build time.
- The Noto Sans JP font is downloaded during `make chrome-assets`, so the build machine needs network access. For reproducible builds, pin the font to a specific upstream revision or commit instead of following the moving Google Fonts main branch.

## 4. Japanese-font smoke test

After deploying, open a page containing Japanese text (for example, a Japanese Wikipedia article) and check:
1. Hiragana, katakana, common kanji, punctuation, and bold text render as glyphs rather than empty boxes.
2. The page can still render Latin text with the existing Liberation Sans font.
3. Reload the application and confirm the font asset request / chrome asset archive completes without a missing-file error.

Bundling a font makes it available to Gecko's font scanner; actual glyph fallback should still be checked in the built runtime.
