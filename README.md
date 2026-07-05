# Terra Incognita

A globe-spinning geography game. A luminous antique-style Earth spins in the
browser; stop it, mark a country (or a U.S. state), and name it. Reach
**100 points** to chart the whole world.

It's a single, self-contained `index.html` — no build step, no backend, no
package install. All map geometry is embedded directly in the file, so it
works offline once loaded and can be hosted anywhere that serves static files.

## How to play

1. The globe spins on load. **Click it** (or press **Stop**) to halt the spin.
2. **Click a country or U.S. state** to mark it — it glows coral and pulses
   as your target, and a countdown timer starts.
3. **Type its name** and press **Name it** (or **Enter**) before time runs out.
   - Correct: **+5 points**, it turns green.
   - Wrong (or the timer expires): **−1 point**, it stays shaded red and the
     real name is revealed.
4. Each target resolves on a single guess (with an extra hint-and-retry step
   on Easy — see below); pick another and keep going until your score
   reaches **100**.

### Difficulty

Pick **Easy**, **Medium**, or **Hard** above the globe at any time — switching
resets the current game. Each tracks its own **best score** and **fewest
rounds to 100**, saved locally in your browser.

| Difficulty | Time to answer | Behavior |
| --- | --- | --- |
| Easy | 30s | Shows a continent/region hint; a wrong first guess gives an extra hint and one free retry instead of resolving the round. |
| Medium | 20s | Straight name guess, no hints. |
| Hard | 10s | Name it correctly, then name its **capital** for full credit (+5); naming the capital wrong after a correct name still earns partial credit (+3). The capital step itself is untimed. |

### Controls

| Action | Control |
| --- | --- |
| Stop / restart the spin | Click the globe, or the **Spin** / **Stop** buttons |
| Rotate the globe | Drag (while stopped) |
| Zoom in / out | Mouse wheel / two-finger scroll |
| Submit a guess | **Enter** or the **Name it** button |

Answers are forgiving: case, accents, punctuation, and a leading "the" are
ignored, and common variants are accepted (e.g. `USA`, `UK`, `UAE`,
`Ivory Coast`, `Burma`, `DR Congo`, or a state's postal code like `CA`, `TX`).

## Run it locally

Just open `index.html` in any modern browser. That's it.

If your browser is strict about local files, serve the folder over HTTP instead:

```bash
# Python 3
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Deploy to GitHub Pages

```bash
git init
git add .
git commit -m "Add Terra Incognita globe game"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo>.git
git push -u origin main
```

Then on the repo: **Settings → Pages → Build and deployment**, set **Source** to
"Deploy from a branch," choose the branch you pushed to (**main**, or whatever
you named it) / **/(root)**, and save. Your game goes live at
`https://<your-username>.github.io/<repo>/` within a minute or so. (This repo
itself deploys from `beta`.)

Because the entry file is named `index.html`, it loads at the root URL
automatically. The same folder also deploys as-is on Netlify, Vercel, and
Cloudflare Pages.

## Tech & credits

- Rendering: [D3](https://d3js.org/) geo (orthographic projection on canvas),
  loaded from a CDN.
- Map data: [Natural Earth](https://www.naturalearthdata.com/) 1:110m admin-0
  countries (public domain), plus U.S. state boundaries from
  [PublicaMundi's public GeoJSON dataset](https://github.com/PublicaMundi/MappingAPI),
  simplified and embedded.
- Type: Fraunces, Space Grotesk, and Space Mono via Google Fonts.

## License

Project code is released under the MIT License — see [LICENSE](LICENSE).
Natural Earth data is in the public domain.
