---
name: create-design-system
description: Create or update a brand-specific design system as a versioned, standalone HTML file in _docs/design-system/. Use for design guidelines, visual tokens, and interactive component showcases with light and dark themes.
---

# Create Design System

Generate a polished design reference at `_docs/design-system/[version-number]-design-system.html`, relative to the target project root. It must open directly in a browser without a server or build step. Creating this reference does not include changing the application's UI or installing a component framework.

## Discover the brand

Inspect project instructions, existing design-system versions, brand assets, fonts, and theme tokens first. Use supplied guidelines and conversation context as the source of truth.

For a new system with missing details, ask one concise, bundled set of questions:

- What is the brand or product, who is it for, and what should it feel like?
- Are there preferred colors, typography, logos, or existing brand guidelines?
- What kind of product is this, and are there particular components or visual references to include?

Offer “not decided—choose for me” as a valid answer. Ask only about missing information. If the user already asked you to choose everything, proceed immediately; otherwise wait for their response before generating the final artifact. Silence does not mean they have delegated brand choices.

When choices are delegated, select a coherent visual direction suited to the known audience and product. Use an explicitly provisional brand name if none exists. Document chosen colors, fonts, personality, and a short rationale in the HTML, distinguishing inferred decisions from supplied requirements. Do not invent an approved logo or brand history.

Prefer Google Fonts for new font choices, while respecting an existing brand typeface. Choose families and weights for the brand, readability, and supported scripts rather than always using the same pairing. Include robust system fallbacks.

## Version and update

- Respect a requested version and the project's established numeric naming convention. Treat the version as a filename segment, never as an arbitrary path.
- Otherwise start at `1.0.0`. For updates, increment the highest existing version numerically, not lexicographically or by modification time. With semantic versions, use a patch for corrections, a minor for additions, and a major for an intentional incompatible token or visual overhaul.
- Create a new version on updates and preserve earlier files unless the user explicitly requests editing in place. Never silently overwrite a different existing version. Choose the next unused version when no exact version was requested; clarify an explicit version collision.
- Read the existing HTML before updating it. Preserve decisions outside the requested changes. Ask further brand questions only when the update needs information that is genuinely missing.
- Propagate changed tokens through previews, documented values, snippets, and both themes. Show the brand, version, date, and a concise change summary in the document.

## Visual direction

Use Material Design and Material UI as references for semantic roles, elevation, states, component anatomy, and accessible interaction. Adapt typography, shape, density, and visual rhythm to the brand. Implement the showcase with native HTML, CSS, and JavaScript; no MUI React runtime is needed. Look up current guidance when a specific design or behavior question requires it.

Treat user references as inspiration, not mandatory layouts or brand palettes. The desired reference patterns include prominent typography specimens, tonal color ramps, and organized component matrices showing sizes, styles, and states. Do not turn reference annotations, dashed design-tool boundaries, style counts, or sample colors into mandatory output features.

Create a modern documentation experience with a branded introduction, clear section navigation, expressive typography, and well-composed previews. A desktop sidebar or sticky index may collapse into accessible mobile navigation. Make the composition feel specific to the brand rather than reproducing a plain reference sheet or applying identical cards and decoration everywhere.

Include a realistic composed product example so users can judge how the system works together. Use brand-relevant copy and sample data. Each component group needs a visual preview, concise usage guidance, relevant token names, variants, and states. Label static state specimens clearly; controls presented as interactive must work.

## Foundations and tokens

Define reusable CSS custom properties for colors and typography, plus spacing, radii, borders, elevation, motion, and layout. Separate primitive values from semantic roles, and consume semantic variables throughout the page and components. Keep literal color and typography values in token definitions. Generate or derive displayed values and snippets from the same definitions so documentation stays accurate.

Include these foundations:

- **Brand:** Audience, personality, visual principles, and the rationale for chosen colors and typography. Show supplied logo usage when available.
- **Colors:** Primary, secondary, neutral, and appropriate accent ramps; success, warning, error, and information roles; backgrounds, surface tiers, text hierarchy, borders, focus, actions, and paired foregrounds for colored surfaces. Label swatches with token names, values, and usage.
- **Typography:** Heading, body, and optional monospace families; display, heading, title, body, label, and caption specimens. Document size, weight, line height, letter spacing, and purpose. Include realistic long text and links alongside large letter specimens.
- **Geometry and layout:** Spacing scale and base unit, padding and gaps, radii, borders, shadows/elevation, layering, content widths, grids, gutters, and breakpoints. Show visual examples with values.
- **Icons and motion:** Consistent icon size and stroke/fill approach, accessible naming, duration/easing tokens, transition examples, and reduced-motion behavior.

Provide selectable CSS token snippets or a token export derived from the definitions. If offering clipboard copy, handle failure under local-file restrictions and provide selectable text instead of falsely reporting success.

## Component coverage

Use the following as the baseline for a new system. Group related patterns to keep the document navigable, and extend coverage for the product or explicit requests. On a narrow update, complete the requested changes without unnecessarily rebuilding every section.

| Group | Examples |
| --- | --- |
| Buttons | Primary/filled, secondary/tonal, outline, text, destructive, and icon-only; sizes and leading/trailing icons; default, hover, focus, pressed, disabled, and loading states |
| Badges and chips | Status badges, counts, filled/outlined tags, selectable and removable chips; appropriate sizes and semantic colors |
| Forms | Text, email, password with visibility toggle, search, textarea, select, checkbox, radio group, switch, slider, and relevant date/file inputs; required, helper, populated, disabled, read-only where supported, success, and error states |
| Form composition | A realistic labeled form with grouped fields, local validation, inline errors, and success feedback without sending data |
| Disclosure | Working accordions, expandable details, and tabs with selected and focus states |
| Navigation | Links, header/sidebar, breadcrumbs, pagination, and responsive navigation with local destinations or local state changes |
| Containers | Cards, dividers, lists, avatars/avatar groups, and a composed example |
| Data display | Table with headers, aligned data, status, row actions, and empty state; working sorting/filtering if those controls are shown |
| Feedback | Alerts, dismissible toast/notification, tooltip, linear/circular progress, loading indicator, skeleton, and empty/error states |
| Overlays | Working dialog/modal and menu or popover, including dismissal and keyboard behavior |
| Product patterns | One composed screen or section for the brand's domain; add relevant pricing, onboarding, course, commerce, or dashboard patterns |

For large matrices, use labeled static specimens alongside a live example. Do not imply functionality with dead controls. Specialized irrelevant widgets can be omitted with a short explanation; always include the explicitly requested buttons, badges, accordions, and forms.

## Light and dark themes

Define complete semantic palettes for both themes. Theme the entire document, including navigation, native form controls, demos, code blocks, overlays, borders, feedback, and interaction states. Do not simulate dark mode with an inversion filter or background-only change.

Provide an obvious keyboard-accessible theme toggle with an accurate accessible name and state. Initialize from a saved choice when available, then `prefers-color-scheme`, then a documented default. Apply the initial root theme before visible content paints and set `color-scheme` to match. Use a brand-scoped storage key and guard storage reads and writes with `try/catch`; local-file storage may be unavailable, and the toggle must still work without persistence.

## Standalone HTML and interaction

- Use one HTML document with a doctype, language, viewport metadata, descriptive title, embedded CSS, and embedded JavaScript. Avoid package installs, runtime imports, CDN JavaScript, and fetching local templates or JSON.
- Inline essential icons as SVG and embed essential visual assets. Component meaning must not depend on remote images or icon fonts.
- Google Fonts may be an optional network enhancement with `display=swap` and fallback stacks. The document must remain readable and interactive offline. If exact offline typography is requested, embed appropriately licensed fonts; otherwise explain that loading the Google font needs connectivity.
- Keep content, anchor navigation, and native controls useful without JavaScript where practical. Demo submissions validate locally without reloads or network requests.
- Prefer native buttons, inputs, `details`/`summary`, and `dialog` where suitable. Give every control an accessible name. Inputs need persistent visible labels, associated hints/errors, and intentional button types.
- Use actual disabled semantics; expose loading/busy states. Do not communicate status or errors through color alone. Preserve entered values on validation failure and announce feedback.
- Tabs need tab/tabpanel relationships, selected state, roving focus, and arrow/Home/End navigation. Otherwise use ordinary anchor navigation rather than tab semantics.
- Dialogs move focus inside, contain modal focus, close with Escape and a close button, and restore focus to the trigger. Menus/popovers need suitable semantics, keyboard behavior, dismissal, and focus restoration.
- Tooltips work on focus as well as hover and are dismissible. Keep essential explanations available without them. Toasts use an appropriate live region and allow important feedback to be read.
- Make focus indicators visible on every surface. Keep sticky navigation clear of anchor targets, prefer roughly 44 CSS pixel touch targets for primary controls, and respect reduced motion. Avoid page-wide horizontal scrolling; tables/code can scroll inside labeled containers.

## Verify and deliver

Check the generated artifact, not just its source. With browser tooling, open the actual file URL and inspect desktop and narrow mobile layouts in both themes. Exercise the theme toggle, accordion, form, tabs, overlays, and other live demos. Check keyboard navigation, focus, reduced motion, console errors, and behavior with network disabled. Verify persistence where storage is supported. For updates, recheck affected components and shared tokens.

Check actual foreground/background pairs in both themes, including muted text, hover, and errors. Meet WCAG AA contrast targets: 4.5:1 for ordinary text, 3:1 for large text, and 3:1 for required control boundaries and state indicators. Do not invent contrast measurements or claim a complete accessibility audit from visual inspection.

Check unique IDs, valid internal links, labels, missing assets, and unfinished placeholders. If browser tooling is unavailable, perform available structural checks and explicitly identify visual and interaction verification as outstanding. Do not claim checks you did not run.

Return a clickable link to the generated HTML, its version, a brief account of the chosen direction or updates, and meaningful verification limits. Explain that it opens directly in a browser; deployment is unnecessary.
