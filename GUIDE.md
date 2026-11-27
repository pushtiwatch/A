# Pipeline guide

How the transcripts and summaries in this repository are produced. The steps are written so that a person, or any capable language model, can follow them. This guide does not name specific models, vendors, or operators.

The aim is to surface what speakers allege, and how serious those allegations are, with as little time and cost as possible. Readability and public reach matter more than word-perfect translation. Frame every claim as the speaker's allegation, and prefer a direct allegation over a softened one.

## Rules

- Published files should list only agreed information: the video, the speaker's words, the English rendering, and the scores and flags derived from them.
- Do not put into the repository any name, path, machine, credential, or model that would identify an operator or a local computer.
- Do not rewrite published history, and do not force a branch update, without an explicit request.
- Do not add a second inventory entry for a `video_id` that is already present.
- Do not re-score or rewrite a video that is already published, unless asked.
- Do not publish a social-media post about these files without explicit approval.

## Overview

Work in batches of about ten videos, highest public view count first. For each video:

1. Download original-language captions.
2. Score short windows of those captions and keep only the highest-ranking lines.
3. Translate only those lines, plus a title, a short summary, and a handful of claims.
4. Check every timestamp, then write the markdown file and the inventory entry.

Scoring first typically cuts the text that must be translated to about 5–10% of the caption file.

## 1. Download captions

Use a public caption downloader (for example `yt-dlp`) to fetch the original-language automatic captions. Prefer `gu-orig` for Gujarati and `hi-orig` for Hindi. If the downloader fails on modern YouTube pages, retry with a JavaScript runtime enabled (`--js-runtimes node`).

Store, for each video, the caption lines with their timestamps and a small metadata file (id, title, language, duration, published date, view count). If a download run is interrupted, check what already finished before starting over; most videos are often already done.

## 2. Rank windows before any translation

Do not translate the full transcript first. A cheap scoring model ranks short windows of the original caption text against the individual flags listed in [`taxonomy.json`](taxonomy.json).

A practical setup used in this repository:

- Windows of six caption lines.
- A window is kept when its score is at least **0.4**.
- From the kept windows, take the title plus about the top 24 caption lines.

Write, per video:

- A short file of lines to translate, beginning with the instruction block in [Translation instructions](#translation-instructions), then `TITLE_ORIGINAL:` and the ranked `[timestamp] original text` lines.
- A skeleton used later to assemble the full transcript (which lines were selected, their order, and the original text).
- A record of ranked windows with scores and flags, used later for **Top allegations** and for the inventory fields `flags`, `explosive_score`, and `claims`.

`first_hand` is a score booster, not a harm flag, but it may still appear in the flags list when the speaker is describing what they say happened to them or what they themselves saw or heard.

## 3. Pack groups

Combine 2–3 videos into one file so a single translation pass covers several items. Put a shared header at the top, then one block per video:

```
There are N separate videos below. Each starts with a line '=== VIDEO <id> ==='. For EACH video, output the line '=== VIDEO <id> ===' and then follow that video's own instructions exactly. Write each claim on its own line starting with 'CLAIM: '. Output nothing else: no intro, no commentary, no markdown formatting, no tables. Plain text only.

=== VIDEO <id> ===
<that video's instruction block and ranked lines>
```

Keep a group small enough that the model's entire reply can be produced in one shot.

## 4. Translate

Use any capable translation model, through a web interface or an API. A person can paste the file, or a browser-automation agent can attach it. Some models refuse this material, or omit timestamps; if that happens, switch to another capable model and redo the group. Do not name the model in any file that will be published.

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

If any timestamp is missing or changed, or a line is dropped, redo the whole group. Do not patch a partial reply by hand unless the only defect is the bare `CLAIM:` header described above.

## 6. Build the markdown file and the inventory entry

Each published video is one markdown file plus one object in [`inventory.json`](inventory.json).

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

`meta` holds `version`, `last_updated`, `total_entries`, a short disclaimer, and `source` (this repository's URL).

Each entry has:

| Field | Content |
| --- | --- |
| `video_id` | YouTube id (unique in the file) |
| `youtube_url` | Canonical watch URL |
| `title_gu` | Original title (Gujarati or Hindi text) |
| `title_en` | English title |
| `language` | `gu` or `hi` |
| `published_date` | `YYYY-MM-DD` |
| `duration_seconds` | Length |
| `location` | List of places mentioned, or `["not stated"]` |
| `summary_en` | Short attributive summary |
| `claims` | List of 3–5 short sentences, each beginning `Alleges that ...` |
| `tags` | Human-readable tags from the taxonomy |
| `flags` | List of **only the flags that apply** (a flag that is absent is false). The full vocabulary lives once in [`taxonomy.json`](taxonomy.json). |
| `explosive_score` | Numeric score from the ranked windows |
| `keywords` | Search terms, often with the original-script form |
| `md_url` | Blob URL of the markdown file on the default branch |

After writing the files, check that no `video_id` is repeated and that the new files contain no operator names, local paths, machine identifiers, credentials, or model names. Then commit with the repository's public identity and publish the default branch.

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

- Some models refuse this material, drop a timestamp, or omit a line. Verify every timestamp against the input. If a line is missing, redo the group with a different capable model.
- Send the whole group in one shot. Bounding the input to the ranked windows (about two dozen lines per video, two or three videos per group) keeps the reply short enough to finish in a single response.
- Scoring can be done for many videos ahead of time; translation groups can then be run one after another. One translation session at a time is more reliable than several in parallel.
- Neutral title prompts produce bland headings such as "Remarks on...". If the files will be used in public posts, ask for a short, specific, allegation-led title.
- An interrupted caption download usually leaves most videos already done. Check progress before repeating work.
