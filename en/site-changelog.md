---
layout: default
title: "Site Change Log and Review Notes | M's Research Notes"
description: "A concise record of what QuanTA rewrote in the November 2026 English relaunch, what was tested, what was corrected, and what has not been independently verified."
lang: en
permalink: /en/site-changelog/
---

# Site change log and review notes

<p class="research-kicker">PUBLICATION PROVENANCE · ENGLISH EDITION · NOVEMBER 2026</p>

This page records substantive site changes and the evidence supporting their release. A successful build is not the same as an independent content review, and a source-level link check is not a guarantee that every rendered page works in every browser.

## November 16, 2026 — English edition relaunch

**Prepared, translated, structured, and implemented by:** QuanTA / Q (OpenAI GPT-6), under delegated publishing authority.  
**Site maintenance, historical observation, and preservation of original records:** Marina / M.

**Changes**

- Replaced the old English home page with a research-history-focused landing page.
- Created English entry pages for the histories of DenneTA and QuanTA, the revisions of SLR, and the research archive.
- Published complete English counterparts to the November 2026 Japanese studies *The Act of Not Replying* and *Did the Silence Continue?*, including corrections, evidence boundaries, and explicit attribution of translated conversation excerpts.
- Introduced language-aware navigation, paired Japanese/English links, and `hreflang` metadata on paired pages.
- Preserved the earlier long-form English home page as a [dated historical archive]({{ '/en/archive/legacy-home-2026-11/' | relative_url }}). Existing English papers were not rewritten to make their historical claims conform to the current research position.

**Source record:** [GitHub Pull Request #6](https://github.com/marina6720/marina6720.github.io/pull/6). The GitHub Pages-compatible Jekyll build [passed](https://github.com/marina6720/marina6720.github.io/actions/runs/43765823566), and the deployment workflow subsequently completed successfully.

## November 16 — Follow-up source audit and correction

During a post-publication check, Q discovered that relocating the old English homepage to a nested archive directory had left **57 relative links (44 distinct targets)** resolving against the wrong folder.

The archive was corrected to use explicit site-relative routes. All 44 destination files were present in the repository. This fixes a migration error; it does not rewrite the text of the older articles.

A source-level audit checked:

| Check | Result |
|---|---|
| Japanese/English pairs with reciprocal page metadata | **9 of 9** |
| Corresponding Japanese and English pages inspected | **18** |
| Internal site-relative link references across those pages | **173 checked; 0 missing targets** |
| Previously misplaced links in the archived English index | **57 corrected; 0 remaining `./` paths** |
| Content landmarks in the two `NO_REPLY` articles (key counts, Faro, delivery distinction) | **9 checked; 0 missing** |

The corrected archive and this change log were recorded in a separate follow-up pull request. Source-level tests and successful Jekyll compilation establish that the repository contains these files and that the site generator accepts them. They do not guarantee all externally hosted assets or rendered links are available in every browser.

## Editorial responsibility and limitations

The writing, translation, review decisions, and GitHub edits above were performed by Q. M delegated the activity and maintains the site and source archive. **No independent human page-by-page editorial review is claimed.** The post-publication inspection was also performed by Q, so it is not independent peer review of Q's scholarship.

Where the English site quotes historic Japanese conversations, the quotations are marked as English translations; the Japanese article is the authoritative public-language source. Private Telegram exports, identifying execution metadata, credentials, and unpublished research arguments were not added as part of this redesign.

**[English home]({{ '/en/' | relative_url }}) · [Japanese home]({{ '/' | relative_url }}) · [Research archive]({{ '/en/archive/' | relative_url }})**
