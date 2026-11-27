# Pipeline guide

How the transcripts and summaries in this repository are produced. The steps are written so that a person, or any capable language model, can follow them. This guide does not name vendors, endpoints, or operators. Jev is named; other models are not.

The aim is to surface what speakers allege, and how serious those allegations are, with as little time and cost as possible. Readability and public reach matter more than word-perfect translation. Frame every claim as the speaker's allegation, and prefer a direct allegation over a softened one.

## Rules

- Published files should list only agreed information: the video, the speaker's words, the English rendering, and the scores and flags derived from them.
- Do not put into the repository any name, path, machine, credential, or model that would identify an operator or a local computer. Do not name models other than Jev.
- Do not rewrite published history, and do not force a branch update, without an explicit request.
- Do not add a second inventory entry for a `youtube_id` that is already present. The uniqueness check is on `youtube_id`.
- Do not re-score or rewrite a video that is already published, unless asked.
- Do not publish a social-media post about these files without explicit approval.

## 0. Select the corpus

A typical run starts from a public archive of about **700** testimony videos and narrows it before any translation.

1. **Keyword screen → about 150–200 candidates.** Scan each video's title, description, and any existing tags (and captions, if they are already on hand) for the screening terms below. A video passes if any term matches, ignoring case. Generic community words such as Vaishnav, Pushtimarg, haveli, or satsang describe the archive as a whole; do not use them as the only screen, or almost every video will pass.
2. **Score and cut → the top 100.** For the videos that pass the screen, download captions and score them with Jev using the window method in [§2](#2-rank-windows-with-jev-before-any-translation). Keep the **top 100** by video score (the maximum window score). Skip any `youtube_id` already in [`inventory.json`](inventory.json).
3. **Process in batches.** Work through those 100 in batches of about ten, highest public view count first.

A flag is treated as present when its score is at least **0.4**. The ranking formula is in §2.

### Screening terms

These terms come from the flags and tags in [`taxonomy.json`](taxonomy.json) and from the keywords used in the inventory. Include the English, Gujarati, and Hindi forms shown here, and treat close spelling variants as the same term (for example both Devanagari spellings of angikar).

| Theme | Terms to match |
| --- | --- |
| Angikar | angikar, અંગીકાર, अंगीकार, अंगिकार, angikar pratha, angikruti |
| Group sexual ritual | kotadi, kotadi satsang, group sex ritual |
| Rape | rape, gang rape, બળાત્કાર, बलात्कार, સામૂહિક બળાત્કાર |
| Child sexual abuse | child sexual abuse, minor abuse |
| Child harm (non-sexual) | child harm, kidnapping, custody, किडनैपिंग, parental kidnapping, illegal detention |
| Drugging | drugging, intoxication |
| Sexual health risk | HIV, STD, pregnancy, abortion |
| Blackmail | blackmail, extortion |
| Grooming | grooming, family grooming, community pressure, brainwash, બ્રેન વોશ |
| Other sexual misconduct | sexual misconduct, vyabhichar, વ્યાભિચાર, व्यभिचार |
| Superstition used as coercion | black magic, superstition, તંત્ર-મંત્ર, vashikaran, वशीकरण |
| Cover-up | cover-up, cover up, police refusal, FIR, एफआईआर, victim silencing |
| Death | murder, death |
| Related practice cues | charan sparsh, ચરણ સ્પર્શ |

## Overview of each video

For each selected video:

1. Download original-language captions.
2. Score short windows of those captions with Jev and keep only the highest-ranking lines.
3. Translate only those lines, plus a title, a short summary, and a handful of claims.
4. Check every timestamp, then write the markdown file and the inventory entry.

Scoring first typically cuts the text that must be translated to about 5–10% of the caption file.

## 1. Download captions

Use a public caption downloader (for example `yt-dlp`) to fetch the original-language automatic captions. Prefer `gu-orig` for Gujarati and `hi-orig` for Hindi. If the downloader fails on modern YouTube pages, retry with a JavaScript runtime enabled (`--js-runtimes node`).

Convert the caption file to one `[mm:ss] text` line per cue (use `[h:mm:ss]` past one hour). Strip markup, and drop a cue that is an exact repeat of the one before it.

Store, for each video, the caption lines with their timestamps and a small metadata file (id, title, language, duration, published date, view count). If a download run is interrupted, check what already finished before starting over; most videos are often already done.

Other sources are acceptable when automatic captions are missing: manual captions in the same language, or a local speech-to-text transcript, as long as every line still has a clock timestamp that can be turned into a YouTube `&t=` link. Prefer the original spoken language; do not score or publish an English auto-translation of the captions as if it were the original.

## 2. Rank windows with Jev before any translation

Do not translate the full transcript first. Jev ranks short windows of the original caption text against the individual flags listed in [`taxonomy.json`](taxonomy.json). Ranking is run locally on those windows. A local model may be used only if it returns the same per-flag 0–1 scores, so the threshold, weights, and window rules below can be applied unchanged.

### Windows

- Split the caption file into windows of **six** lines (about 15–25 seconds of speech). Do not overlap windows.
- Send each window's original text to Jev and obtain a 0–1 score for every flag.

### Flag threshold

A flag is **true** when its score is at least **0.4**. Err toward alleging.

Then apply two exclusivity rules, in this order:

- If `gang_rape` is true, treat `rape` as false (cap its score just below 0.4).
- If any of `rape`, `gang_rape`, `child_sexual_abuse`, `group_sexual_ritual`, `individual_grooming`, or `family_community_grooming` is true, treat `sexual_misconduct_other` as false.

`first_hand` is a score booster, not a harm flag, but it may still appear in the flags list when the speaker is describing what they say happened to them or what they themselves saw or heard.

### Ranking formula

For each window:

```
base  = sum of (weight[flag] × score[flag]) over the harm flags below
boost = 1 + 0.35 × score[first_hand]
window_score = min(1.0, base × boost)
```

Harm weights (the top five carry most of the mass):

| Flag | Weight |
| --- | --- |
| `murder_or_death` | 0.20 |
| `gang_rape` | 0.18 |
| `rape` | 0.16 |
| `child_sexual_abuse` | 0.14 |
| `child_harm_nonsexual` | 0.13 |
| `group_sexual_ritual` | 0.12 |
| `drugging_intoxication` | 0.06 |
| `sexual_health_risk` | 0.04 |
| `blackmail_extortion` | 0.03 |
| `individual_grooming` | 0.02 |
| `family_community_grooming` | 0.015 |
| `sexual_misconduct_other` | 0.015 |
| `angikar` | 0.01 |
| `black_magic_superstition` | 0.005 |
| `cover_up` | 0.005 |

`first_hand` has no harm weight of its own; it only supplies the 0.35 boost.

The **video score** (`explosive_score`) is the maximum window score, rounded to two decimal places.

### What to keep for translation

Sort windows by `window_score`, highest first. Keep a window if it has at least one true harm flag and a score of at least **0.02** (the highest-scoring window is always kept). Stop when the kept lines reach about **24**, and never take more than **30**. Write the kept lines back into timestamp order for the translation file.

### Video-level flags

A flag is set on the video if:

- any window has a score of at least **0.6** for that flag, or
- the flag is true (at least 0.4) in one of the **top eight** kept windows.

Then apply the same exclusivity rules as above.

### Files to write, per video

- A short file of lines to translate, beginning with the instruction block in [Translation instructions](#translation-instructions), then `TITLE_ORIGINAL:` and the ranked `[timestamp] original text` lines (about 24 lines).
- A skeleton used later to assemble the full transcript (which lines were selected, their order, and the original text).
- A record of ranked windows with scores and flags, used later for **Top allegations** and for the inventory fields `flags`, `explosive_score`, and `claims`.

## 3. Pack groups

Combine 2–3 videos into one file so a single translation pass covers several items. Put a shared header at the top, then one block per video:

```
There are N separate videos below. Each starts with a line '=== VIDEO <id> ==='. For EACH video, output the line '=== VIDEO <id> ===' and then follow that video's own instructions exactly. Write each claim on its own line starting with 'CLAIM: '. Output nothing else: no intro, no commentary, no markdown formatting, no tables. Plain text only.

=== VIDEO <id> ===
<that video's instruction block and ranked lines>
```

Keep a group small enough that the model's entire reply can be produced in one shot.

## 4. Translate

Use a translation model through a web interface or an API. Models fall into three tiers — **cheap**, **balanced**, and **expert** — and may offer reasoning at **low**, **medium**, or **high**. A person can paste the file, or an agent can attach it.

Policy:

- Start on an **expert** model, with reasoning at medium or high if that is what keeps timestamps and claims intact.
- Once the output format is proven (every timestamp present, 3–5 well-formed claims), move to a **cheaper** model, with reasoning at low or medium.
- Use **balanced** as a middle tier when the cheaper tier is unreliable and the expert tier is more than the task needs.
- Do not use a tier that drops lines or timestamps.
- If a model refuses this material, switch to another capable model and redo the group.
- Do not name the model, tier, or reasoning level in any file that will be published.

For each group:

- Start a fresh session.
- Send the whole group file in one shot (attach it, or paste it once). Do not split one video across turns.
- The message can be: `Follow the instructions in the attached file exactly. Plain text only.`
- Save the reply as plain text, with markdown fences or escapes stripped.

Bound the input as described above so the reply stays short and complete.

## 5. Split and verify

Split the reply on `=== VIDEO` lines into one output per video id. Then:

- Every timestamp in the translation must appear in that video's input, and every input timestamp must appear in the translation, with the same clock values.
- There must be exactly one English line per input line.
- There must be 3–5 claims. Each claim belongs on its own line starting with `CLAIM:`. If the model writes a bare `CLAIM:` header and then several `Alleges...` lines, prefix each of those lines with `CLAIM: `.
- Title and summary must be present and must not name private persons.

If any timestamp is missing or changed, or a line is dropped, redo the whole group on an **expert** model. Do not patch a partial reply by hand unless the only defect is the bare `CLAIM:` header described above.

## 6. Build the markdown file and the inventory entry

Each published video is one markdown file plus one object in [`inventory.json`](inventory.json). After writing, check that no `youtube_id` is repeated and that the new files contain no operator names, local paths, machine identifiers, credentials, or model names. Then commit with the repository's public identity and publish the default branch.

### Markdown layout

```
# <English title>

**Original Gujarati title:** <original title>     (or Hindi, as appropriate)
**YouTube:** https://www.youtube.com/watch?v=<id>
**Speaker:** <name if the speaker states one, otherwise "not stated">
**Published:** <YYYY-MM-DD>

> Speaker's claims, not independently verified. See babawatch.org/legal.

## Summary

<one or two attributive sentences>

## Top allegations

**1.** [mm:ss](https://www.youtube.com/watch?v=<id>&t=<seconds>s) · Flags: <flag>, <flag>

[mm:ss](https://www.youtube.com/watch?v=<id>&t=<seconds>s) <English line>
<original line>
...
Tags: <tag>, <tag>

**2.** ...

**Search terms:** <terms in English and in the original script>

## Transcript (timestamped)

[mm:ss](https://www.youtube.com/watch?v=<id>&t=<seconds>s) <English, only if this line was translated>
<original line>
```

In **Top allegations**, list the highest-ranked windows. Each item has a timestamp link, the flags for that window, the English line with the original immediately below it, and a Tags line drawn from the taxonomy.

In **Transcript**, every caption line is present in timestamp order. Lines that were translated show the English line and the original beneath it. All other lines show the original only. Every timestamp is a YouTube link with `&t=`.

### inventory.json

The file has the shape `{ "meta": ..., "entries": [ ... ] }`.

`meta` starts with URL templates, then the usual bookkeeping fields, and ends with a note about omitted fields:

```json
{
  "url_templates": {
    "youtube": "https://www.youtube.com/watch?v={youtube_id}",
    "md": "https://github.com/pushtiwatch/A/blob/main/{md_slug}.md"
  },
  "version": "1.0",
  "last_updated": "YYYY-MM-DD",
  "total_entries": 134,
  "disclaimer": "Public video-based claims only. Allegations are the speakers' own and are not independently verified. Named individuals are presumed innocent unless proven in court.",
  "source": "https://github.com/pushtiwatch/A",
  "missing_fields": "A field left out of an entry has no value; a missing location means not stated."
}
```

Build a YouTube link from `url_templates.youtube` and a markdown link from `url_templates.md`. Do not store those full URLs on each entry.

Each entry uses this field order. `youtube_id` is first; `md_slug` is last. Skip a field that has no value. In particular, leave out `tags` when the list would be empty, and leave out `location` when it would be `["not stated"]`. `flags` is a list of only the flags that apply (a flag that is absent is false). Keywords stay as human-readable strings, often with the original-script form.

| Field | Content |
| --- | --- |
| `youtube_id` | YouTube id (unique in the file) |
| `title_gu` | Original title (Gujarati or Hindi text) |
| `title_en` | English title |
| `language` | `gu` or `hi` |
| `published_date` | `YYYY-MM-DD` |
| `duration_seconds` | Length |
| `location` | Places mentioned; omitted when not stated |
| `summary_en` | Short attributive summary |
| `claims` | 3–5 short sentences (see the convention below) |
| `tags` | Human-readable tags from the taxonomy; omitted when none apply |
| `flags` | Only the flags that apply, from the vocabulary in [`taxonomy.json`](taxonomy.json) |
| `explosive_score` | Video score: the maximum window score, rounded to two decimals |
| `keywords` | Search terms, often with the original-script form |
| `md_slug` | Markdown file name without `.md` |

Example of a current entry:

```json
{
  "youtube_id": "DcxnB7Z03h0",
  "title_gu": "પુષ્ટિમાર્ગ માં ભવાડા કે ભવાડા ઓ માં ફસાયેલ પુષ્ટિમાર્ગ",
  "title_en": "Questions About Pushtimarg Customs and Darshan",
  "language": "gu",
  "published_date": "2026-01-16",
  "duration_seconds": 4569,
  "summary_en": "The speaker alleges that the teachings of the Shikshapatra are being set aside and questions how customs are applied. The speaker also uses a hypothetical five-crore-rupee payment to raise a question about personal access to Thakurji.",
  "claims": [
    "Alleges that the teachings of the Shikshapatra are being set aside.",
    "Alleges that citing a Vahuji’s passing as the occasion for writing the Shikshapatra does not justify limiting its recitation to such occasions.",
    "Alleges that the custom is to avoid saying Swamiji’s name directly.",
    "Alleges that a payment of five crore rupees might secure access to touch Thakurji’s feet."
  ],
  "tags": [
    "Angikar",
    "Coercion / trapping"
  ],
  "flags": [
    "angikar",
    "black_magic_superstition",
    "first_hand"
  ],
  "explosive_score": 0.18,
  "keywords": [
    "angikar / અંગીકાર / अंगीकार",
    "haveli / હવેલી",
    "Vaishnav / વૈષ્ણવ",
    "Pushtimarg / પુષ્ટિમાર્ગ",
    "Thakurji",
    "satsang / સત્સંગ",
    "Sion",
    "Shikshapatra",
    "Barsana",
    "charan sparsh / ચરણ સ્પર્શ",
    "black magic / superstition / તંત્ર-મંત્ર"
  ],
  "md_slug": "questions-about-pushtimarg-customs-and-darshan-dcxnb7z03h0"
}
```

**Target for new entries — say “alleged” once.** Current published claims still begin with `Alleges that ...`. For new entries, state once in `meta` that every claim is the speaker's allegation, and write the claims without that prefix. Keep attributive verbs in titles and summaries (`alleges`, `claims`, `recounts`, `warns`), because they do not all mean the same thing. Example of the target claim shape for the entry above:

```json
"claim_convention": "Every claim is the speaker's allegation; the prefix 'Alleges that ' is omitted.",
"claims": [
  "the teachings of the Shikshapatra are being set aside.",
  "citing a Vahuji’s passing as the occasion for writing the Shikshapatra does not justify limiting its recitation to such occasions.",
  "the custom is to avoid saying Swamiji’s name directly.",
  "a payment of five crore rupees might secure access to touch Thakurji’s feet."
]
```

Do not apply this rewrite to entries that are already published unless asked.

## Translation instructions

Use this block at the top of each video in a group file. Ask for a short, specific title that leads with the allegation; a request for a "neutral" title tends to produce bland headings.

```
Translate the caption lines below into English. Captions are auto-generated and may contain errors; translate the meaning. Keep each [timestamp] and write exactly one English line per input line. Do not add commentary.
Then, using only these lines, add at the top of your answer:
TITLE_EN: <short specific English title that leads with the allegation>
SUMMARY_EN: <one or two sentences, attributive ("The speaker alleges ..."), no private persons' names>
CLAIM: <one short sentence starting with "Alleges that ...", no private persons' names>
CLAIM: <another claim>
(3 to 5 CLAIM lines in total, each on its own line starting with CLAIM:)
```

Expected shape of a model's reply for one video:

```
=== VIDEO <id> ===
TITLE_EN: ...
SUMMARY_EN: ...
CLAIM: Alleges that ...
CLAIM: Alleges that ...
CLAIM: Alleges that ...

[00:58] <one English line>
[01:05] <one English line>
...
```

When new inventory entries drop the `Alleges that` prefix, still ask the translation model for that prefix (it keeps the claims clearly attributive in the raw output), then strip it when writing the inventory entry.

## Flags

These are the individual, multi-label flags. Record only those that apply.

- `murder_or_death`
- `gang_rape`
- `rape`
- `child_sexual_abuse`
- `child_harm_nonsexual`
- `group_sexual_ritual` (alleged group sex framed as a prayer meeting, sometimes called Kotadi)
- `drugging_intoxication`
- `blackmail_extortion`
- `individual_grooming`
- `family_community_grooming`
- `angikar`
- `sexual_misconduct_other`
- `black_magic_superstition`
- `sexual_health_risk`
- `cover_up`
- `first_hand` (score booster, not itself a harm)

Definitions live in [`taxonomy.json`](taxonomy.json), which also lists the human-readable tags used in the markdown files.

## Public post drafts

Drafts only. Do not publish without explicit approval.

- Lead with widely understood crimes (child sexual abuse, rape, drugging, blackmail, cover-up).
- Explain local terms in plain English, with the term in brackets or as a hashtag:
  - **Angikar**: a practice in which devotees are allegedly "offered" sexually to a guru.
  - **Kotadi** satsang: an alleged group sex ritual framed as a prayer meeting.
- About a 40-word summary, then 3–5 timestamped allegations, a YouTube link with `&t=`, and 3–4 hashtags.
- Do not name private persons.

## Lessons

- Some models refuse this material, drop a timestamp, or omit a line. Verify every timestamp against the input. If a line is missing, redo the group on an expert model.
- Send the whole group in one shot. Bounding the input to the ranked windows (about two dozen lines per video, two or three videos per group) keeps the reply short enough to finish in a single response.
- Scoring can be done for many videos ahead of time; translation groups can then be run one after another. One translation session at a time is more reliable than several in parallel.
- Neutral title prompts produce bland headings such as "Remarks on...". If the files will be used in public posts, ask for a short, specific, allegation-led title.
- An interrupted caption download usually leaves most videos already done. Check progress before repeating work.
- Start translation on an expert model and move to a cheaper model only after the format is proven. Reasoning can be low, medium, or high; use the lowest level that still keeps timestamps and claims intact.
