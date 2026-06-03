
---
name: seo-skills
description: Enterprise-grade React SEO architecture and technical SEO optimization skill
---

# Implementation Plan: Dynamic SEO & Editor Collapsible Fixes

**Version:** 1.0



---

## 1. Problem Statement

| # | Problem | Impact |
|---|---------|--------|
| 1 | Admin editors save hardcoded fallback values instead of `null` | Dynamic SEO fallback on visitor page never triggers |
| 2 | Advanced SEO fields are always visible — clutters editor UI | Poor UX for non-technical content editors |
| 3 | `mergeSeoDefaults` has no contract or test coverage | Silent SEO regressions in production |
| 4 | No canonical URL strategy defined | Duplicate content indexed by Google |
| 5 | JSON-LD field has no validation before save | Broken structured data silently |
| 6 | OG image has no dimension/format validation | Broken social previews |

---

## 2. Null Contract — Single Source of Truth

**Rule:** All SEO override fields must save `null` (not `""`) when left empty.  
Apply this in every `saveX()` function across all editors.

```js
// helpers/seoSanitize.js — shared utility
export const sanitizeSeoPayload = (fields) => {
  const out = {};
  for (const [key, val] of Object.entries(fields)) {
    out[key] = typeof val === 'string' && val.trim() === '' ? null : val ?? null;
  }
  return out;
};
```

**Usage in every editor save function:**
```js
// Before (broken)
seo_title: form.seo_title || 'Default Site Title',

// After (correct)
import { sanitizeSeoPayload } from '../helpers/seoSanitize';
const seoFields = sanitizeSeoPayload({
  seo_title:       form.seo_title,
  seo_description: form.seo_description,
  og_title:        form.og_title,
  og_description:  form.og_description,
  og_image:        form.og_image,
  twitter_card:    form.twitter_card,
  twitter_title:   form.twitter_title,
  twitter_image:   form.twitter_image,
  schema_json:     form.schema_json,
  canonical_url:   form.canonical_url,
});
```

---

## 3. Files to Modify

### 3.1 `helpers/seoSanitize.js` — NEW FILE

- `sanitizeSeoPayload(fields)` — converts `""` to `null`
- `validateSchemaJson(str)` — returns `{ valid, error }`
- `buildCanonical(type, slug)` — returns absolute canonical URL

```js
const BASE_URL = 'https://yourdomain.com'; // env var in production

export const buildCanonical = (type, slug) => {
  const routes = {
    insight:    `/insights/${slug}`,
    career:     `/careers/${slug}`,
    'case-study': `/case-studies/${slug}`,
    service:    `/services/${slug}`,
    industry:   `/industries/${slug}`,
  };
  const path = routes[type] ?? `/${slug}`;
  return `${BASE_URL}${path}`; // no trailing slash — enforced here
};

export const validateSchemaJson = (str) => {
  if (!str || str.trim() === '') return { valid: true, error: null };
  try {
    JSON.parse(str);
    return { valid: true, error: null };
  } catch (e) {
    return { valid: false, error: e.message };
  }
};
```

---

### 3.2 `admin/style.css` — MODIFY

Add collapsible styles. Scope tightly to avoid conflicts with existing `<details>` elements.

```css
/* SEO Collapsible Accordion */
details.editor-collapsible-details {
  border: 1px solid var(--border-color, #e2e8f0);
  border-radius: 8px;
  margin-top: 1.5rem;
  overflow: hidden;
}

details.editor-collapsible-details > summary {
  list-style: none;
  cursor: pointer;
  padding: 0.875rem 1rem;
  font-weight: 500;
  font-size: 0.9rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  user-select: none;
}

details.editor-collapsible-details > summary::-webkit-details-marker {
  display: none;
}

details.editor-collapsible-details > summary::after {
  content: '›';
  font-size: 1.1rem;
  transition: transform 0.2s ease;
  display: inline-block;
}

details.editor-collapsible-details[open] > summary::after {
  transform: rotate(90deg);
}

details.editor-collapsible-details > .collapsible-body {
  padding: 1rem;
  border-top: 1px solid var(--border-color, #e2e8f0);
}

/* Filled-fields badge on summary */
details.editor-collapsible-details > summary .seo-badge {
  font-size: 0.7rem;
  background: #dbeafe;
  color: #1e40af;
  padding: 2px 8px;
  border-radius: 999px;
  font-weight: 400;
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  details.editor-collapsible-details {
    border-color: #334155;
  }
  details.editor-collapsible-details > summary .seo-badge {
    background: #1e3a5f;
    color: #93c5fd;
  }
}

/* Narrow viewport */
@media (max-width: 640px) {
  details.editor-collapsible-details > summary {
    font-size: 0.85rem;
    padding: 0.75rem;
  }
}
```

---

### 3.3 `admin/insights.html` — MODIFY

**A. Wrap SEO block in collapsible:**

```html
<details class="editor-collapsible-details" id="seo-collapsible">
  <summary>
    Advanced SEO
    <span class="seo-badge" id="seo-badge" style="display:none">0 fields set</span>
  </summary>
  <div class="collapsible-body">
    <!-- Open Graph fields -->
    <!-- Twitter Card fields -->
    <!-- Schema JSON-LD field (with inline validator) -->
  </div>
</details>
```

**B. JSON-LD inline validation:**

```html
<textarea id="schema_json" rows="6" placeholder='{"@context":"https://schema.org",...}'></textarea>
<p id="schema-error" style="color:red;font-size:0.8rem;display:none"></p>

<script>
document.getElementById('schema_json').addEventListener('blur', function () {
  const { valid, error } = validateSchemaJson(this.value);
  const el = document.getElementById('schema-error');
  el.style.display = valid ? 'none' : 'block';
  el.textContent = valid ? '' : `Invalid JSON: ${error}`;
});
</script>
```

**C. Filled-fields badge:**

```js
function updateSeoBadge() {
  const seoFields = ['og_title','og_description','og_image',
    'twitter_title','twitter_image','schema_json','seo_title','seo_description'];
  const filled = seoFields.filter(id => {
    const el = document.getElementById(id);
    return el && el.value.trim() !== '';
  }).length;
  const badge = document.getElementById('seo-badge');
  if (filled > 0) {
    badge.style.display = 'inline';
    badge.textContent = `${filled} field${filled > 1 ? 's' : ''} set`;
  } else {
    badge.style.display = 'none';
  }
}
// Call on load + on each input change inside the collapsible
```

**D. Fix `saveArticle()` payload:**

```js
async function saveArticle() {
  // Validate JSON-LD before proceeding
  const schemaVal = document.getElementById('schema_json').value;
  const { valid, error } = validateSchemaJson(schemaVal);
  if (!valid) {
    alert(`Schema JSON is invalid: ${error}. Please fix before saving.`);
    return;
  }

  const canonical = buildCanonical('insight', form.slug);

  const payload = {
    title:           form.title,
    content:         form.content,
    // ... other core fields ...

    // SEO overrides — always null if empty, never hardcoded fallbacks
    ...sanitizeSeoPayload({
      seo_title:       form.seo_title,
      seo_description: form.seo_description,
      og_title:        form.og_title,
      og_description:  form.og_description,
      og_image:        form.og_image,
      twitter_card:    form.twitter_card || 'summary_large_image', // safe hardcoded default
      twitter_title:   form.twitter_title,
      twitter_image:   form.twitter_image,
      schema_json:     schemaVal,
      canonical_url:   canonical, // always generated, never null
    }),
  };

  await api.saveArticle(payload);
}
```

---

### 3.4 `admin/careers.html`, `case-studies.html`, `services.html` — MODIFY

Same collapsible pattern. Same `sanitizeSeoPayload` in each `saveX()`. Same JSON-LD validator. Same badge.

```html
<!-- Pattern is identical across all three -->
<details class="editor-collapsible-details">
  <summary>
    Advanced SEO
    <span class="seo-badge" id="seo-badge" style="display:none"></span>
  </summary>
  <div class="collapsible-body">
    <!-- OG / Twitter / Schema fields -->
  </div>
</details>
```

Change only the `buildCanonical` type argument per editor:
- careers → `buildCanonical('career', slug)`
- case-studies → `buildCanonical('case-study', slug)`
- services → `buildCanonical('service', slug)`

---

### 3.5 `admin/index.html`, `admin/industries.html` — MODIFY

Align existing Advanced SEO blocks to use `.editor-collapsible-details` class structure. No logic changes — layout/class alignment only.

---

### 3.6 `InsightDetailPage.jsx` — MODIFY

**Enforce `mergeSeoDefaults` contract:**

```js
// utils/mergeSeoDefaults.js

const BASE_URL = 'https://yourdomain.com';
const REQUIRED_FIELDS = ['title', 'description', 'canonical', 'ogImage', 'twitterCard'];

export function mergeSeoDefaults(article) {
  const slug = article.slug ?? '';

  const meta = {
    title:       article.seo_title       || article.title,
    description: article.seo_description || article.excerpt || '',
    canonical:   `${BASE_URL}/insights/${slug}`, // always absolute, never from DB
    ogTitle:     article.og_title        || article.seo_title || article.title,
    ogDescription: article.og_description || article.excerpt || '',
    ogImage:     article.og_image        || article.featured_image || `${BASE_URL}/default-og.jpg`,
    twitterCard: article.twitter_card    || 'summary_large_image',
    twitterTitle:  article.twitter_title || article.title,
    twitterImage:  article.twitter_image || article.og_image || article.featured_image,
    schemaJson:  article.schema_json     || buildDefaultArticleSchema(article),
  };

  // Dev-only contract check
  if (process.env.NODE_ENV === 'development') {
    REQUIRED_FIELDS.forEach(f => {
      if (!meta[f]) console.warn(`[SEO] mergeSeoDefaults: missing field "${f}" for slug "${slug}"`);
    });
  }

  return meta;
}

function buildDefaultArticleSchema(article) {
  return JSON.stringify({
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: article.title,
    description: article.excerpt || '',
    image: article.featured_image || '',
    datePublished: article.published_at || '',
    dateModified: article.updated_at || '',
    author: {
      '@type': 'Person',
      name: article.author_name || 'Nexero Team',
    },
  });
}
```

**In `InsightDetailPage.jsx`:**

```jsx
import { mergeSeoDefaults } from '../utils/mergeSeoDefaults';

export default function InsightDetailPage() {
  const { slug } = useParams();
  const article = useArticle(slug); // your data hook

  const seo = mergeSeoDefaults(article);

  return (
    <>
      <Helmet>
        <title>{seo.title}</title>
        <meta name="description" content={seo.description} />
        <link rel="canonical" href={seo.canonical} />

        <meta property="og:title"       content={seo.ogTitle} />
        <meta property="og:description" content={seo.ogDescription} />
        <meta property="og:image"       content={seo.ogImage} />
        <meta property="og:url"         content={seo.canonical} />
        <meta property="og:type"        content="article" />

        <meta name="twitter:card"       content={seo.twitterCard} />
        <meta name="twitter:title"      content={seo.twitterTitle} />
        <meta name="twitter:image"      content={seo.twitterImage} />

        <script type="application/ld+json">{seo.schemaJson}</script>
      </Helmet>

      {/* page content */}
    </>
  );
}
```

---

## 4. Database — Schema Requirements

All SEO columns must be nullable with no default value:

```sql
ALTER TABLE articles
  MODIFY seo_title       VARCHAR(255)  NULL DEFAULT NULL,
  MODIFY seo_description TEXT          NULL DEFAULT NULL,
  MODIFY og_title        VARCHAR(255)  NULL DEFAULT NULL,
  MODIFY og_description  TEXT          NULL DEFAULT NULL,
  MODIFY og_image        VARCHAR(512)  NULL DEFAULT NULL,
  MODIFY twitter_card    VARCHAR(50)   NULL DEFAULT NULL,
  MODIFY twitter_title   VARCHAR(255)  NULL DEFAULT NULL,
  MODIFY twitter_image   VARCHAR(512)  NULL DEFAULT NULL,
  MODIFY schema_json     TEXT          NULL DEFAULT NULL;

-- canonical_url is always computed — do not store in DB
```

> Apply the same pattern for `jobs`, `case_studies`, `services`, `industries` tables.

---

## 5. Canonical URL Strategy

| Rule | Value |
|------|-------|
| Protocol | Always `https` |
| www | Never — `yourdomain.com` only |
| Trailing slash | Never |
| Source | Always computed in code via `buildCanonical()` — never read from DB |
| Redirect | 301 `http://` → `https://`, `www.` → non-www at server/CDN level |

---

## 6. Twitter Card Default

Always fall back to `summary_large_image` — never leave this field empty or `null` in the rendered `<meta>` tag.

```js
twitterCard: article.twitter_card || 'summary_large_image',
```

---

## 7. Unit Tests

File: `__tests__/mergeSeoDefaults.test.js`

```js
import { mergeSeoDefaults } from '../utils/mergeSeoDefaults';

const base = {
  slug: 'my-article',
  title: 'My Article',
  excerpt: 'A short excerpt.',
  featured_image: 'https://yourdomain.com/img/hero.jpg',
};

test('falls back to article title when seo_title is null', () => {
  const meta = mergeSeoDefaults({ ...base, seo_title: null });
  expect(meta.title).toBe('My Article');
});

test('uses seo_title when set', () => {
  const meta = mergeSeoDefaults({ ...base, seo_title: 'SEO Title Override' });
  expect(meta.title).toBe('SEO Title Override');
});

test('canonical is always absolute and has no trailing slash', () => {
  const meta = mergeSeoDefaults(base);
  expect(meta.canonical).toBe('https://yourdomain.com/insights/my-article');
  expect(meta.canonical.endsWith('/')).toBe(false);
});

test('twitter card defaults to summary_large_image', () => {
  const meta = mergeSeoDefaults({ ...base, twitter_card: null });
  expect(meta.twitterCard).toBe('summary_large_image');
});

test('ogImage falls back to featured_image', () => {
  const meta = mergeSeoDefaults({ ...base, og_image: null });
  expect(meta.ogImage).toBe('https://yourdomain.com/img/hero.jpg');
});

test('ogImage falls back to default-og.jpg when no image exists', () => {
  const meta = mergeSeoDefaults({ ...base, og_image: null, featured_image: null });
  expect(meta.ogImage).toBe('https://yourdomain.com/default-og.jpg');
});

test('buildDefaultArticleSchema is valid JSON when schema_json is null', () => {
  const meta = mergeSeoDefaults({ ...base, schema_json: null });
  expect(() => JSON.parse(meta.schemaJson)).not.toThrow();
});
```

---

## 8. Verification Checklist

### Build
- [ ] `npm run build:web` — zero compilation errors
- [ ] `npm test` — all 7 SEO unit tests pass

### Admin UI
- [ ] SEO collapsible opens and closes without losing input values
- [ ] Filled-fields badge shows correct count when SEO fields are populated
- [ ] Invalid JSON in Schema field shows inline error and blocks save
- [ ] SEO section renders correctly at 320px viewport width

### DB → Frontend Sync
- [ ] Save article with all SEO fields blank → DB stores `null` for all SEO columns
- [ ] Visitor page renders correct fallback `<title>`, OG, and canonical from article data
- [ ] Update article title in DB → visitor page `<title>` reflects the new title (no hardcoded stale value)
- [ ] Canonical URL format: `https://yourdomain.com/insights/[slug]` — no trailing slash, no `www`

### SEO Validation
- [ ] Paste canonical URL into [Google Rich Results Test](https://search.google.com/test/rich-results)
- [ ] Paste page URL into [Facebook Sharing Debugger](https://developers.facebook.com/tools/debug/)
- [ ] Confirm `twitter:card` meta is always `summary_large_image` — never missing

---

## 9. Implementation Order

```
1. helpers/seoSanitize.js          — create shared utility (no dependencies)
2. utils/mergeSeoDefaults.js       — update with contract + tests
3. __tests__/mergeSeoDefaults.test.js — write and pass all tests
4. admin/style.css                 — add collapsible styles
5. admin/insights.html             — primary editor (null fix + collapsible + validator + badge)
6. admin/careers.html              — collapsible + null fix
7. admin/case-studies.html         — collapsible + null fix
8. admin/services.html             — collapsible + null fix
9. admin/index.html                — class alignment only
10. admin/industries.html          — class alignment only
11. InsightDetailPage.jsx          — wire updated mergeSeoDefaults
12. DB migration                   — alter SEO columns to NULL DEFAULT NULL
13. Full verification checklist
```

---

## 10. Risk Register

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Existing articles have `""` instead of `null` in DB | High | One-time migration: `UPDATE articles SET seo_title = NULL WHERE seo_title = ''` |
| `mergeSeoDefaults` called with `undefined` article | Medium | Add null guard at top of function |
| Canonical mismatch between computed and sitemap | Medium | Generate sitemap URLs using same `buildCanonical()` utility |
| Editor saves before JSON-LD validation fires | Low | Validate in `saveArticle()` entry point, not just on `blur` |
