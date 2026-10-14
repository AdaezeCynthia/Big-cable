# BCM Developer Test — Submission

**Candidate:** Ndu Adaeze Cynthia

This repository contains my answers to the ten-question BCM developer test. Code
files are organised by question; written answers are below. Setup steps and the
checks I ran are included for each code question.

> **Note on screenshots:** placeholders are referenced below (e.g. `Q5-validation.png`).
> I have run each piece locally and will add the screenshots into each question
> folder as evidence.

---

## How to run the code answers

- **Q4 (CSS layout):** open `Q4-css-layout/index.html` in Chrome. Resize the window
  (or use DevTools device mode, Ctrl+Shift+M) to see the mobile and desktop layouts.
- **Q5 (HTML):** open `Q5-html/index.html` in Chrome. Validated with the W3C Markup
  Validator (validator.w3.org). A local `test.jpeg` is referenced by the image.
- **Q6 (JavaScript):** open `Q6-javascript/index.html` in Chrome. Click **Hit me**
  to add paragraphs; click **Toggle image paragraphs** to hide/show image paragraphs.
- **Q1 (PHP):** `Q1-php/posts-loop.php` is a WordPress theme fragment. It runs inside
  a WordPress theme (verified by code review / local WordPress install).

---

## AI disclosure

I used AI tools to help identify bugs, structure solutions, and check my reasoning.
I verified every answer myself: the frontend answers (Q4, Q5, Q6) by running them
locally in the browser and testing each required case; Q5 against the W3C validator;
and the PHP and written answers by working through the logic and, where noted,
testing locally. I can explain all of it and make changes live.

---

# PHP

## Q1 — Posts loop (corrected)

See `Q1-php/posts-loop.php` for the corrected code.

**Bugs found and their effects:**

1. **Wrong pagination key.** The original used `'page' => $page`. For a `WP_Query`
   posts loop the correct key is **`'paged'`**. With `'page'`, pagination silently
   fails — every page shows the same first 10 posts.
2. **Undefined / unsafe variables.** `$pages` was never defined (should be
   `$q->max_num_pages`), and `$_GET['s']` was read without checking it exists, which
   throws a warning in PHP 8 and was not safely handled.
3. **Broken regex.** `'#[ad]#'` is a regex character class — it matches a single
   `a` or `d`, not the literal string `[ad]`. The brackets must be escaped:
   `'#\[ad\]#'`.
4. **`preg_replace_callback` misuse.** The callback used `echo` and referenced an
   undefined `$m[1]`. `preg_replace_callback` must **return** the replacement string,
   and the callback never received a matches argument. As written it wiped the content.
5. **Missing `wp_reset_postdata()`** after the custom loop, which can break other
   queries on the page.

**Checks (how to verify):**

- **Missing parameters:** load with no query string — defaults to page 1, empty search.
- **Page 2:** `?page=2` returns posts 11–20 (works now that `'paged'` is correct).
- **No search results:** `?s=somethingnonexistent` shows a "No results" message.
- **Multiple [ad] markers:** a post body with two or more `[ad]` markers produces
  `ad-1`, `ad-2`, … via the static counter.

**Assumption:** `[ad]` is a literal placeholder authors type into content; a separate
JavaScript handler later replaces each `.ad` div with a live ad. This code only
outputs uniquely-identifiable, empty ad containers.

---

# WordPress admin

## Q2 — An editor cannot upload images, but a colleague can

**Key clue:** one user can upload and another cannot, so the cause is almost certainly
**user-specific**, not site-wide (a site-wide problem like disk space or folder
permissions would break uploads for everyone).

**Likely causes, in the order I would check them (most likely / cheapest first):**

1. **User role / capabilities.** Uploading requires the `upload_files` capability.
   The affected user may be a **Contributor** (which cannot upload by default), or a
   role-management plugin may have altered their capabilities.
   - *Evidence:* compare both users' roles under **Users**; check for a role plugin
     that has changed capabilities.
   - *Fix:* grant `upload_files` (assign the correct role, e.g. Author/Editor, or
     restore the capability). Verify by having the editor upload an image.

2. **Browser / client-side issue.** Could be their browser — a blocking extension
   (ad-blocker/privacy tool), corrupted cache/session, or an outdated browser.
   - *Evidence:* try an incognito window with extensions disabled, or another
     browser/device; check the browser console for errors during upload.
   - *Fix:* if incognito works, disable the offending extension or clear cache.

3. **File-specific / limits.** The specific file may exceed the max upload size or be
   an unsupported MIME type.
   - *Evidence:* read the exact error — "exceeds maximum upload size" points to
     `upload_max_filesize`; "file type is not permitted" points to MIME restrictions.
     Ask whether *any* image fails or only this one.
   - *Fix:* raise the limit, or allow/convert the file type.

**Ruled out early:** server-side `wp-content/uploads` folder permissions — unlikely
here, because another user uploading successfully proves the folder is writable.

---

## Q3 — Article page is slow, especially on mobile

I measure a baseline before changing anything.

**Three improvements, each with what I would measure first:**

1. **Optimise images.** (Publishing sites are image-heavy; unoptimised images are the
   most common mobile bottleneck.)
   - *Measure first:* run Lighthouse / PageSpeed Insights on mobile throttling and
     check the Network tab for total image weight and the largest images; check for
     modern formats (WebP/AVIF) and lazy-loading.
   - *Fix:* responsive images (`srcset`), modern formats, `loading="lazy"` below the fold.

2. **Reduce / defer JavaScript, especially third-party scripts** (ads, analytics,
   embeds).
   - *Measure first:* Lighthouse "Total Blocking Time" and "Reduce the impact of
     third-party code"; Performance tab for script-execution time.
   - *Fix:* defer/async non-critical scripts; load ads/analytics without blocking;
     remove unused scripts.

3. **Caching (page cache + CDN).**
   - *Measure first:* check Time To First Byte (TTFB) in the Network tab / WebPageTest;
     a high TTFB points to server/database work caching would fix. Check whether a
     cache plugin/CDN is already active.
   - *Fix:* enable page caching and a CDN for static assets.

**Chosen improvement — image optimisation — and how I'd verify it helped without
breaking ads, analytics, or signed-in behaviour:**

I'd take a before/after Lighthouse measurement on mobile throttling, comparing
Largest Contentful Paint and total page weight for a concrete number. Image
optimisation doesn't touch ads, analytics, or auth logic, which is why it's a safe
first choice — but I'd still confirm: check ad slots still render (and exclude ad
containers from lazy-loading), confirm analytics still fires by watching network
requests, and test the page both signed-out and signed-in. **Caching is the risky one
for signed-in users** — you can accidentally serve a cached logged-out page to a
logged-in user — which is another reason I verify image optimisation first and treat
caching more carefully.

---

# Troubleshooting and delivery

## Q7 — Some readers see an old version of a corrected article, others see the update

**First three checks:**

1. **Is it caching?** Check whether affected readers are served by a CDN/page cache.
   Compare the raw origin response (bypass cache, e.g. a cache-busting query string or
   direct origin request) against what cached users see. If origin is correct but some
   users see stale content, it's a **caching** problem.
2. **Is it code?** Check what actually deployed — the release commit/build. Confirm the
   corrected content is present in the deployed code/template, and that the deploy
   completed on all servers (not a partial rollout where some nodes have old code).
3. **Is it content?** Check the article record itself (revision history) — confirm the
   correction was actually saved/published and isn't sitting in a draft/revision, and
   isn't duplicated across two records.

**Distinguishing caching from code/content:** if the origin serves the correct version
and only some users see stale content, it's caching (purge and re-verify). If the
origin itself serves the old version, it's code (wrong/partial deploy) or content (the
fix wasn't saved/published).

**When I'd roll back:** if the new release caused the inconsistency (e.g. a broken
deploy serving old templates) and a cache purge doesn't fix it, roll back to the last
known-good release. If it's purely a cache issue, purge rather than roll back.

**Verifying recovery:** purge caches, then check the article from multiple locations /
a fresh session / mobile and desktop, confirming everyone now sees the corrected
version; confirm the origin and cached responses match.

**Two-sentence update to the editor:** "We've identified that some readers were served
a cached older version of the article; we've purged the cache and confirmed the
corrected version now shows for everyone. We're monitoring to ensure it stays
consistent, and will follow up if anything changes."

*Assumption:* a CDN/page cache sits in front of the site (common for a publishing
platform), rather than guessing the specific cause.

---

# Initiative and judgment

## Q8 — Editors manually copy titles, links, and images into several tools

**What I'd ask/observe first:** which exact tools they copy into, how often, and where
mistakes actually happen — ideally by watching one editor do it once. That tells me
which step is the real pain and whether it's worth automating.

**Smallest useful improvement:** a single "share preview" block on the article (or a
small internal page) that shows the title, link, and image already formatted and
ready to copy with one click each — removing the retyping that causes mistakes, while
keeping editors in full control of what they paste where.

**Simpler alternative with little/no new code:** a saved template/snippet (even a
documented copy-paste checklist, or a browser text-expander) that standardises the
format, so editors copy consistent content without a new feature.

**Why this choice:** it targets the actual error source (manual retyping) with the
least build, and keeps publishing decisions with editors rather than automating them
away.

**What remains under editorial control:** which tools to post to, when, and any edits
to the copy — I only make the correct content easy to grab.

**Testing with one editor:** have one editor use it for a few real articles, compare
time taken and mistakes against their current process, and get their feedback.

**Judging success:** fewer copy errors and less time per article, plus the editor
preferring it over the old way.

*Similar improvement I've made (under 100 words):* At Fowgate I noticed the same
validation and error-handling bugs recurring across different modules, each fixed in
isolation. Rather than keep patching them, I built reusable form-validation and
error-handling patterns applied consistently across modules. The outcome was more
stable behaviour and fewer recurring defects. What I learned: fixing the pattern once,
at the root, beats repeatedly fixing the symptom — and a small shared improvement can
remove a whole category of problems rather than one instance of it.

---

# APIs and data reliability

## Q9 — Newsletter signup: handle timeout, rate limit, and invalid email without
duplicates

**Approach:** make the request **idempotent** so a retry can't create a duplicate.
Generate a stable idempotency key (e.g. derived from the normalised email) and send it
with the request; the server (or a local dedupe record) uses it to recognise a repeat.
Validate the email before sending. Treat a timeout as *unknown*, not *failed* —
reconcile before blindly retrying.

```text
function subscribe(email):
    email = normalise(email)                 # trim + lowercase

    if not isValidEmail(email):
        return { status: "invalid_email" }   # reject before any network call

    key = idempotencyKey(email)               # stable key for this signup

    # If we already recorded success for this key, do not send again.
    if store.hasSucceeded(key):
        return { status: "already_subscribed" }

    try:
        res = http.post("/subscribe",
                        body={ email },
                        headers={ "Idempotency-Key": key },
                        timeout=5s)

        if res.status == 200 or res.status == 201:
            store.markSucceeded(key)
            return { status: "subscribed" }

        if res.status == 429:                 # rate limited
            retryAfter = res.header("Retry-After") or backoff(key)
            scheduleRetry(key, after=retryAfter)   # retry later, same key
            return { status: "queued_rate_limited" }

        if res.status in 400..499:
            return { status: "rejected", code: res.status }

        # 5xx: server error, safe to retry with same idempotency key
        scheduleRetry(key, after=backoff(key))
        return { status: "queued_retry" }

    catch Timeout:
        # Unknown outcome: the server may have accepted it.
        # Do NOT blindly retry as a new request. Reconcile first, then
        # retry with the SAME idempotency key so a duplicate can't be created.
        scheduleReconcile(key, email)
        return { status: "pending_unknown" }
```

**What I'd store/log:** the idempotency key, normalised email, and status
(pending/succeeded/failed) with timestamps; retry attempts and outcomes.

**What I'd avoid logging:** nothing sensitive beyond what's needed — avoid logging full
personal data repeatedly; don't log secrets/API keys; store the email once against the
key rather than scattering it through logs.

**Testing a retry after the service already accepted the request:** simulate the
timeout path where the mock service actually recorded the signup, then trigger the
retry with the same idempotency key and assert the final state is a **single**
subscriber record, not two.

---

# Security and code review

## Q10 — Endpoint accepts an article ID + new status from the browser

**Reviewing the claim ("safe because the button is only visible to editors"):** this is
**not safe**. Hiding a button in the UI is client-side only — anyone can call the
endpoint directly (curl, DevTools, a crafted request) regardless of what the browser
shows. Authorisation must be enforced on the **server**, never by UI visibility.

**Server-side checks I would require, in order:**

```text
1. Authentication:   is the request from a logged-in user? (reject if not)
2. Authorisation:    does THIS user have permission to edit THIS article?
                     (role/capability check, e.g. can edit posts / owns it)
3. CSRF protection:  valid anti-CSRF token / nonce present?
4. Input validation: is articleId a valid existing article?
                     is newStatus one of an allowed set (whitelist)?
5. Perform update; return a safe response. Log the action (who/what/when).
```

**Two tests that should be rejected:**

1. A **logged-in non-editor** (e.g. a subscriber) calling the endpoint directly to
   change an article's status — must be rejected at the authorisation step.
2. A request with an **invalid or disallowed `newStatus`** (e.g. a value not in the
   allowed set, or an `articleId` the user doesn't own/that doesn't exist) — must be
   rejected at input validation.

**How I'd raise it with the developer:** privately and constructively — acknowledge the
intent, then explain that UI visibility isn't a security control because the endpoint
is directly reachable, and walk through the checks above together. Framed as "here's
how we make this safe," not blame.

**Must be fixed before release:** server-side authentication + authorisation (per-user,
per-article), CSRF protection, and input validation/whitelisting of status values.
