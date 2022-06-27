---
title: Into the Dark
layout: post
published: true
---

I've finally dragged this blog into the 21st century with the addition of **Dark Mode**. In this post I'll walk through some of the main tasks, and dig into the approach I took.

The first step was to get rid of colours hard-coded into the CSS. The old style-sheet was full of `#FFF`s and `#222`s along with more esoteric tints, tones, and shades. I worked colour by colour to pull them into a single block at the top of the sheet in the `:root` scope:

```css
:root {
  /* COLOURS */
  --col-accent: #16b1c3;
  --col-accent-dimmed: #0d7581;
  --col-light: #FFF;
  --col-light-subtle: #FBFBFB;
  --col-light-dimmed: #999;
  --col-dark: #222;
  --col-dark-subtle: #333;
  --col-dark-dimmed: #666;
}
```

My theme was based around a single accent colour of `#16b1c3`, and a dimmed variant of that colour. The remaining parts of the site were either wites, or off whites, which I classed as 'light' colours; or near blacks which I classed as dark colours. With the colours extracted to the top of the file and their uses replaced with `var(--col-light)` and so on the next step was to introduce a seam of abstraction to separate the concrete light and dark colours from their use in the rest of the style-sheet. For this I introduced aliases for each colour prefixed with either `--col-fg` for foreground (dark) colours, or `--col-bg` for background (light) colours.

```css
:root {
  /* BASE colour scheme, with no preferences or overrides */

  --col-bg: var(--col-light);
  --col-bg-dimmed: var(--col-light-dimmed);
  --col-bg-subtle: var(--col-light-subtle);
  --col-fg: var(--col-dark);
  --col-fg-dimmed: var(--col-dark-dimmed);
  --col-fg-subtle: var(--col-dark-subtle);
}
```

Instead of directly referencing light or dark colours most places in the style were changed to reference a `--col-fg-` or `--col-bg` variant instead:

```css
body {
  background: var(--col-bg) none;
  color: var(--col-fg-subtle);
}
```

Now all that was left was to change the foreground and background colours depending on the user's preference. The first step in this was to respect the preference exposed from the browser. We can query this using a CSS meia query, just as we would do for device width. Given our base colour scheme is light we only need to query to see if the user has a preference for dark colours. If the user has _no_ preference, or prefers lighter colourschemes we're fine to use our default scheme.

```css
/* Apply a dark color scheme */
@media (prefers-color-scheme: dark) {
  :root {
    --col-bg: var(--col-dark);
    --col-bg-dimmed: var(--col-dark-dimmed);
    --col-bg-subtle: var(--col-dark-subtle);
    --col-fg: var(--col-light);
    --col-fg-dimmed: var(--col-light-dimmed);
    --col-fg-subtle: var(--col-light-subtle);
  }
}
```

With that set the site was now dark-mode compatible. It respected the user's stated preference for light or dark colours. We can go one better though and allow the user to change the colour scheme on the fly with JavaScript. I begun by adding a stub HTML element to the site's layout which can be latched onto by the JS to provide progressive enhancement:

```html
<div id="dark-mode-placeholder"></div>
```

The core of the JS then revolves around querying the same `prefers-color-scheme: dark` property we did from CSS, and writing a data property at the root of the document if we want to override it:

```javascript
const prefersDarkMode =
    window.matchMedia("(prefers-color-scheme: dark)");

// Toggle the ligth / dark mode override.
const toggleMode = () => {
    const dataSet = document.documentElement.dataset;
    const currentMode = dataSet.colorMode;

    if (currentMode === "auto" ||
        currentMode === undefined) {
        if (prefersDarkMode.matches) {
            dataSet.colorMode = "light";
        } else {
            dataSet.colorMode = "dark";
        }
        localStorage.setItem(
            "stashed-theme", dataSet.colorMode);
    } else {
        dataSet.colorMode = "auto";
        localStorage.removeItem("stashed-theme");
    }

    return dataSet.colorMode;
}

// Restore the theme from local storage if stashed
const stashedTheme =
    localStorage.getItem("stashed-theme");
if (stashedTheme) {
    document.documentElement.dataset.colorMode =
        stashedTheme;
}

// Add a button to the DOM to allow toggling dark mode.
const darkModeButton = document.createElement("a");
darkModeButton.addEventListener("click", () => {
    toggleMode();
});
darkModePlaceholder.appendChild(darkModeButton);
```

The final step was to make the CSS respect these new colour preference properties set by the JavaScript. To do this we need to handle both the cases where the user _prefers_ light and has toggled to dark, and the user preferring _dark_ and switching to light mode.

```css
/* Dark mode override */
html[data-color-mode="dark"] {
  --col-bg: var(--col-dark);
  --col-bg-dimmed: var(--col-dark-dimmed);
  --col-bg-subtle: var(--col-dark-subtle);
  --col-fg: var(--col-light);
  --col-fg-dimmed: var(--col-light-dimmed);
  --col-fg-subtle: var(--col-light-subtle);
}

/* Light mode  override */
html[data-color-mode="light"] {
  --col-bg: var(--col-light);
  --col-bg-dimmed: var(--col-light-dimmed);
  --col-bg-subtle: var(--col-light-subtle);
  --col-fg: var(--col-dark);
  --col-fg-dimmed: var(--col-dark-dimmed);
  --col-fg-subtle: var(--col-dark-subtle);
}
```

That's it! With the colour variables switched either based on the user's media preference, or via a `data-color-mode` property at the root node dark modification is complete. The last remaining step for this site was to update the styles for code blocks to ensure that colours were legible in both light and dark environments.

I am by no means a frontend expert but I enjoyed the process of tweaking CSS and refactoring it as if it were backend code. I'm happy with the resulting style sheet. I hope my travels prove useful to others.
