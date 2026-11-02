# Plan: Give every page on the site one shared layout

**Problem:** Every page of the site (local dev server: `http://127.0.0.1:8080`) has its own header, footer and page structure. Visitors see different navigation, branding and footers from page to page, and every page has to be edited separately whenever anything shared changes.

**Goal:** One header, one footer, one page skeleton and one stylesheet, all defined once and used by every page. Page content stays exactly as it is today.

---

## Rules for the agent

- Work on a feature branch. Never commit straight to the main branch.
- **Don't change page content** (text, images, forms, page-specific features). Only the shared parts (the header, footer, `<head>`, outer page structure and global CSS) move into the layout.
- Migrate the pages in small batches and commit after each batch, so any one change can be reverted.
- If two pages disagree about something real (a different phone number, a menu link that exists on only one page), **don't guess**. Add it to the list of open questions in Phase 1 and ask the site owner.
- Don't add a framework the site doesn't already use unless the owner approves it (see Phase 2).

---

## Phase 0: Discovery (read only, no changes)

1. **Find out what the site is built with.** Check the project files and the command that serves port 8080:
   - `package.json` (Next.js, React, Vue, Astro, Vite, Express, 11ty and so on)
   - `composer.json` or `*.php` files (plain PHP, Laravel, WordPress theme)
   - `requirements.txt`, `manage.py` or `app.py` (Django, Flask)
   - only `.html` files (a static site, possibly served with `python -m http.server 8080` or `npx serve`)
2. **List every page.** Record each page's URL, the file that renders it, and whether it has a header, footer or nav. A quick way is to crawl from the home page:
   ```bash
   npx -y linkinator http://127.0.0.1:8080 --recurse --format csv > pages.csv
   ```
3. **Take "before" screenshots** of every page at desktop width (1366px) and mobile width (390px) with Playwright, and save them in `audit/before/`.
4. **Compare the pages.** For each page, note the differences:
   - logo, menu items and their order, menu link targets
   - footer content: contact details, social links, copyright year, legal links
   - `<head>`: `<title>`, meta description, favicon, CSS and JS files loaded, fonts, `lang` and `dir` attributes
   - wrapper structure (`<header>`, `<nav>`, `<main>`, `<footer>`, or plain `<div>`s), container widths
   - inline `<style>` blocks and repeated or conflicting CSS
5. **Write it up** in `audit/layout-audit.md`: a table with one row per page and one column per difference above.

**Done when:** every page is listed, the stack is known, and the differences are written down.

---

## Phase 1: Decide on the standard

1. Pick the **best current page** as the starting point, usually the home page. Then write down the standard:
   - **Header:** logo (and its link), the full menu in final order, call-to-action button, language switcher if any, mobile menu behavior.
   - **Footer:** columns, contact details, social links, legal links, and a copyright year that updates itself.
   - **`<head>`:** charset, viewport, a title pattern (`{Page title} | {Site name}`), meta description, favicon, Open Graph tags, one global stylesheet, one global script.
   - **Page structure:** `<body>` → `<header>` → `<main id="main">` → `<footer>`, plus a "skip to content" link.
   - **Design tokens:** colors, fonts, spacing scale and container width, all as CSS variables in one place.
2. If the site is in **Arabic or bilingual**, set `lang`/`dir` (`ar`/`rtl`) in the layout itself, use CSS logical properties (`margin-inline-start` rather than `margin-left`), and check that the header and footer mirror correctly in RTL.
3. Put every open question (conflicting phone numbers, menu links that differ between pages) in `audit/layout-audit.md`, and **get the site owner's answers before Phase 3**.

**Done when:** the standard is written down and the owner has answered the open questions.

---

## Phase 2: Build the shared layout once

Use whatever the site's own stack already provides for shared layouts:

| Stack | How to share the layout |
|---|---|
| Plain HTML (static) | Add a small build step that supports includes, preferably **Eleventy (11ty)**: `_includes/base.njk` holds the head, header and footer, and each page sets `layout: base.njk`. If the owner doesn't want a build step, use server-side includes or, as a last resort, `fetch()` the header and footer into placeholders (worse for SEO, and the page flickers while they load). |
| PHP | `includes/header.php` and `includes/footer.php`, pulled in with `require` on every page. Set the page title and meta tags in a variable before the require. |
| WordPress | Clean up the theme's `header.php`, `footer.php` and `functions.php`, and remove any hard-coded headers from page templates. |
| Django / Flask (Jinja) | `templates/base.html` with `{% block title %}` and `{% block content %}`, and every page template `{% extends "base.html" %}`. |
| Laravel | `resources/views/layouts/app.blade.php` with `@yield('content')`, and pages use `@extends('layouts.app')`. |
| React / Vite | `<Layout>` with `<Header/>` and `<Footer/>`, wrapping the router outlet. |
| Next.js | `app/layout.tsx` (or `pages/_app.tsx`) rendering `Header`, `Footer` and `{children}`. |
| Vue / Nuxt | `layouts/default.vue` with `<slot/>`, or `App.vue` around `<router-view/>`. |
| Astro | `src/layouts/BaseLayout.astro` with `<slot/>`. |

Steps:
1. Create the header and footer components, plus the head/meta component if the stack has one.
2. Create **one** global stylesheet (for example `css/main.css` or `styles/global.css`) holding the design tokens, reset, header, footer and container styles. Page-specific styles go in their own files or stay scoped to the page.
3. Mark the current page in the menu automatically (`aria-current="page"` plus an active class).
4. Use the new layout on **one page first** (the home page) and check that it looks and works the same as before.

**Done when:** the layout exists and the home page uses it without visible changes, except where the standard intentionally changes something.

---

## Phase 3: Move the remaining pages onto the layout

For each page, or each small batch of pages:
1. Delete the page's own header, footer, wrapper and duplicate `<head>` markup.
2. Wrap the remaining page content in the shared layout, and pass the page title and meta description.
3. Move any CSS that is still needed into the global stylesheet if it's shared, or a page stylesheet if it isn't. Remove CSS that duplicates or conflicts with the global stylesheet.
4. Make sure each page has exactly one `<h1>`, and that it sits inside `<main>`.
5. Load the page, compare it with its "before" screenshot, and commit (`refactor(layout): move about/contact pages to shared layout`).

**Done when:** no page has its own copy of the header or footer. A search for header and footer markup should match only the layout files:
```bash
grep -rln "<header\|<footer\|class=\"navbar" --include=*.{html,php,njk,jsx,tsx,vue,blade.php} .
```

---

## Phase 4: Clean up

- Delete CSS and JS files that nothing uses any more, and old copies of the header and footer.
- Use one version of each library (Bootstrap, jQuery, icon fonts and so on) across the whole site.
- Make sure every page loads the same favicon, fonts and analytics snippet, each exactly once.
- Write a short `README` section on how to add a new page, so that a new page gets the layout automatically.

---

## Phase 5: Check everything

1. **Screenshots:** take "after" screenshots (Playwright, desktop and mobile) and compare them with `audit/before/`. Content areas should be unchanged, and headers and footers should now be identical on every page.
2. **Links:** run `npx -y linkinator http://127.0.0.1:8080 --recurse`. It should find no broken links.
3. **HTML:** validate a sample of pages (`npx -y html-validate` or the W3C validator).
4. **Accessibility and SEO:** run Lighthouse on 3–5 pages. Each page should have a unique title and description, `lang`/`dir` set, a skip link, and keyboard-usable navigation including the mobile menu.
5. **Responsive behavior:** check that the mobile menu opens and closes on every page, and that nothing scrolls sideways at 390px.
6. **RTL (if the site has Arabic pages):** check those pages in RTL.

---

## Acceptance criteria

- [ ] The header, footer and `<head>` basics are each defined in **exactly one place**.
- [ ] Every page uses the shared layout, including 404 and error pages.
- [ ] Menu, logo, contact details and copyright are identical on all pages, and the current page is highlighted.
- [ ] There is one global stylesheet with design tokens, and no conflicting duplicate CSS.
- [ ] Page content is unchanged, as shown by the before/after screenshots.
- [ ] No broken links. The HTML validates on the sampled pages. Lighthouse accessibility score is 90 or higher on the sampled pages.
- [ ] The README explains how to add a new page using the layout.
- [ ] The work is on its own branch, committed in small reviewable steps, with the audit (`audit/`) included.

---

## What the agent needs from the site owner

1. Access to the site's source code (the project folder or repository behind `http://127.0.0.1:8080`) and the command that starts it.
2. Answers to the open questions from Phase 1 (the final menu, contact details, footer links).
3. Approval if the plan calls for adding a build tool such as Eleventy to a plain-HTML site.
