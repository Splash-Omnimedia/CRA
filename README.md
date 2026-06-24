# Carolinas Roofing Association (CRA) — Website

Marketing website for the **Carolinas Roofing Association (CRA)**, formerly the Carolinas Roofing & Sheet Metal Contractors Association (CRSMCA). Built and maintained by [Splash Omnimedia](https://splashomnimedia.com/).

## What this is

A single-page, self-contained static site. All markup, styles, and scripts live in `index.html` (inline `<style>` and `<script>` blocks), so there is no build step and no external CSS/JS dependencies beyond Google Fonts. Navigation between "pages" (Home, About, Board Members, Past Presidents, Membership, Events, Directory, Partners, Contact, etc.) is handled client-side via hash routing.

## Structure

```
cra-website/
├── index.html        # The entire site (markup + inline CSS + inline JS)
├── images/           # Photography and partner logos used by the site
├── README.md
└── .gitignore
```

## Run locally

No build tools required. Either open `index.html` directly in a browser, or serve the folder:

```bash
# Python 3
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Deploy

Because it is a static site, it can be hosted anywhere that serves files — GitHub Pages, Netlify, or a standard web host. For GitHub Pages, enable Pages on the `main` branch (root) in the repository settings.

## Notes

- **Rebrand:** the site reflects the CRA rebrand. An intro load screen animates the transition from "CRSMCA" to "CRA" on first load.
- **Pending content:** Board Member headshots and bios show "Bio Coming Soon" placeholders, and the full Past Presidents list is still to be supplied by the client. Internal reminders for these are kept as HTML comments (not visible to visitors).
- **Self-Insurers Fund:** the "CRSMC – Self-Insurers Fund" program name is intentionally retained as the fund's proper name.
