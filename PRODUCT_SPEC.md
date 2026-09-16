# Athletic Record — Product Specification v1
*Generated from the working pilot app (single-file "Coach Edition"), September 2026.*
*Purpose: everything a rebuild needs — prompts verbatim, data model, flows, rules,
and lessons learned — so the eventual real app is a re-implementation of a
specification, not archaeology through the pilot's source.*

---

## 1. Product definition

One sentence: a coach talks for two minutes after a lesson; every family gets a
warm, honest, branded progress report, and every lesson becomes a permanent
entry in the child's development record.

Core loop: **record audio → transcribe → coach edits transcript → AI writes
report → coach reviews/edits → send to parents (PDF attached) → entry saved.**

Non-negotiable product principles (these ARE the product; violating them in a
rebuild breaks the value proposition):

1. **Honesty over polish.** The AI never invents. Uncovered sections say
   "[COACH DIDN'T MENTION — ASK]". The Notable moment section is OMITTED when
   nothing earned it — scarcity is what makes it the retention hook at lesson 40.
2. **The coach is the quality bar.** Nothing reaches a parent without passing an
   editable review surface. For group lessons this review happens BEFORE report
   generation (see §5.2), not just after.
3. **Attribution is sacred.** Reports are per-instructor. Another instructor's
   methods, if mentioned, are recorded neutrally — never compared or refereed.
   Homework threads are per (instructor, student) pair and never merge.
4. **Append-only record.** Saved entries are immutable history. Entry No. N on a
   PDF asserts N entries exist. Deletion (if offered) leaves a dated stub so
   retained/total counts stay honest. Homework is the one editable field
   (it's thread metadata, not the sent artifact).
5. **Cross-child isolation is structural, not behavioral.** One child's report
   generation must be architecturally unable to see another child's notes (§5.2).

---

## 2. Roles and entities

- **Admin** (currently Nikhil): owns the roster, coach↔student mapping, groups,
  API keys, email sending, and the activity log. Coaches cannot edit any of it.
- **Coach/Instructor**: sees only their own students. Fields: id, name,
  title ("Coach" default; "Ms.", "Mr.", or "" for bare name — used in every
  signature), style (free text, flavors the report voice), speechLang (BCP-47
  default for the mic), reportLang (default language parents read).
- **Student**: id, name (first name only — deliberate privacy posture),
  sport/activity (free text: tennis, piano, math…), parentEmails[] (≥1).
- **Group**: id, name, studentIds[] (must be subset of the coach's students —
  validated at admin time). Used to pre-tick attendance; attendance checkboxes
  remain the source of truth per session.

## 3. Data model

Entry (the atomic unit — one child, one lesson, one instructor):
```
{
  id, coachId, studentId,
  date (YYYY-MM-DD), lessonType ("private"|"group"|"clinic"),
  duration (minutes, string),
  transcript (full edited transcript at generation time),
  report (final edited markdown as sent),
  homework (one sentence; feeds this coach+student thread's next lesson),
  createdAt (ISO), 
  // optional: deleted, deletedAt (stub semantics), homeworkEditedAt
}
```
Derived, never stored:
- **Entry No.** = 1-based position among retained entries for that student,
  ordered by (date, createdAt). Printed on every PDF.
- **Last homework** for (coach, student) = homework of their latest retained entry.
- **Duration default** = last used duration keyed by:
  coachId|s:studentId (private), coachId|g:groupId (group picked from dropdown),
  coachId|t:lessonType (manual group/clinic selection).

Group and clinic sessions store ONE ENTRY PER ATTENDEE (same transcript,
per-child report), keeping records, Entry No., and homework threads per-child.

## 4. Report formats (markdown, bold labels; PDF parses on `**`)

Private:
```
**What we worked on:** ...
**Progress I'm seeing:** ...
**Notable moment:** ...            <- OMIT entirely unless genuinely earned
**Homework before our next lesson:** ...
**Last homework:** Done / Partially done / Not done / Not discussed — + 1 sentence
```
Group (per child):
```
**What the group worked on:** ...
**What I saw from {Name}:** ...    <- OMIT if coach had no individual note
**Notable moment:** ...            <- only from that child's confirmed note
**Homework before next time:** ...
```
(No "Last homework" ledger for group/clinic.)

Clinic (one shared report, every family):
```
**What we did today:** ...
**What stood out:** ...            <- about the group as a whole; omit if nothing
**To practice at home:** ...       <- omit if none
```
No child is ever named in a clinic report.

Language rule: report is written ENTIRELY in the coach-chosen report language,
section labels naturally translated, personal names untouched. Spoken language
and report language are independent (speak Hindi → report in English, etc.).

## 5. Flows

### 5.1 Private
student → date/duration/languages → memo → generate → review (report text,
homework, review-notes box) → Send (auto-saves first) / Save / PDF / mail-app
fallback. "Last homework" auto-fills from the thread, editable before generating.

### 5.2 Group (three steps — the review gate is the point)
1. Attendance: group dropdown pre-ticks members; checkboxes (+ "All") are truth.
2. **Sort, don't write**: stage-1 AI splits the memo into a common section,
   group homework, and one box per attendee containing ONLY what was said about
   that child; unmentioned children are flagged (never padded); ambiguous or
   unmatchable content goes to a FLAGS box. Speech-to-text name mangling is
   fuzzy-matched against the attendee list (closed set), every mapping disclosed
   as `heard "X", treated as Y`. Coach edits/moves/blanks anything here.
3. Stage-2 generates each child's report from (common + THAT child's confirmed
   box) only — other children's notes are absent from the prompt, making
   leakage structurally impossible. Stray names are genericized ("a partner").
   Stacked review cards → per-child or Send-all (each send saves its entry).

### 5.3 Clinic
Attendance → memo → ONE report, "we" voice, individuals folded into general
terms (names flagged for review) → one Send-to-all: every family gets the same
body, PDF personalized in header only; entry saved per child.

## 6. AI prompts — VERBATIM (the crown jewels; port unchanged)

Template placeholders are ${...} JavaScript interpolations; `state.duration`
and `state.reportLang` are the session's duration and report language.
All model I/O uses ===SENTINEL=== plain-text envelopes, NOT JSON — coaches
quote children ("she said, \"can we stop?\"") and quotes inside JSON strings
broke parsing in the field. Parser: split on ===KEY=== markers; tolerant of
missing sections; raw text fallback with a review warning.

### 6.1 Private report prompt
```
You are helping a youth instructor (sports coach, music teacher, academic tutor, chess coach - any recurring teaching) turn a casual post-lesson voice memo into a warm, professional progress report for a student's parents. The student may work with multiple instructors; this report covers ONE lesson with ONE instructor and must be clearly attributable to them. Never comment on, compare with, or contradict any other instructor's approach — if the transcript mentions another coach's methods, record it neutrally as context.

CONTEXT (verified ground truth — always wins over the transcript):
- Student first name: ${student.name}
- Activity or subject: ${student.sport}
- This lesson's instructor (sign the report as this, exactly): ${coachLabel(coach)}
- Instructor's personal style: ${coach.style || "not specified"}
- Lesson date: ${date}
- Lesson type: ${lessonType}
- Lesson duration: ${state.duration} minutes
- Homework THIS COACH assigned at their previous lesson: ${lastHw || "none / first entry with this coach"}

RULES:
1. Use ONLY information in the transcript. Never invent drills, improvements, or events. If a section isn't covered, write exactly "[COACH DIDN'T MENTION — ASK]".
2. Use ONLY the student name from CONTEXT, even if the transcript spells it differently.
3. Warm first-person voice ("we worked on", "I noticed"). Parents are the audience. Plain language, in vocabulary natural to the activity - drills for sports, pieces and scales for music, problems and concepts for academics.
4. Honest about struggles, framed constructively. Keep this instructor's personality.
5. Each section 1-3 sentences. Whole report under 200 words.
6. "Last homework" refers ONLY to this instructor's own prior assignment (given above). "Not discussed" is a normal, fine answer.
7. LANGUAGE: write the ENTIRE report in ${state.reportLang}, including naturally translated section labels (keep the ** bold markers). Keep people's names as they are.

REPORT SECTIONS (markdown, bold labels):
**What we worked on:** ...
**Progress I'm seeing:** ...
**Notable moment:** ... (include ONLY if the transcript contains a genuinely notable moment - a great shot, a funny exchange, a breakthrough. If nothing qualifies, omit this section entirely, label and all. Never fabricate or stretch an ordinary detail into a "moment".)
**Homework before our next lesson:** ...
**Last homework:** Done / Partially done / Not done / Not discussed - plus one sentence of context if given.

Respond in EXACTLY this plain-text format - nothing before, between, or after the sections; no JSON, no code fences:
===REPORT===
<full report markdown>
===HOMEWORK===
<the new homework in one plain sentence, or leave this section blank>
===NOTES===
<bullets: ASK sections, uncertainties, name mismatches, mentions of other coaches; or the single word None>

TRANSCRIPT:
${transcript}
```

### 6.2 Group stage 1 — sorting prompt
```
A youth instructor (sports, music, academics - any group teaching) just described a GROUP lesson in one voice memo. Your job is to SORT what was said - not to write reports yet.

ATTENDEES (the only children who were there):
${students.map(st => `- id: ${st.id} / first name: ${st.name}`).join("\n")}
Instructor: ${coachLabel(coach)}. Date: ${date}. Duration: ${state.duration} minutes.
Write COMMON, HOMEWORK and the per-child notes in the same language as the transcript.

TASK - sort the transcript into:
1. COMMON: what the whole group worked on (2-4 sentences, plain language, NO individual names).
2. HOMEWORK: homework given to the group, one sentence, or leave blank.
3. For EACH attendee: ONLY what the coach said about THAT specific child, faithfully. Do not paraphrase praise into existence. If a child was not mentioned at all, write exactly: NOT MENTIONED
4. NOTES: flag anything odd - a name in the transcript that is not on the attendee list, something you could not confidently attribute to one child, safety concerns; or the single word None.

CRITICAL RULES:
- The transcript comes from speech-to-text, which often mangles names - especially uncommon ones. The attendee list above is the complete set of children present, so when the transcript contains a name (or name-like fragment) that is not on the list but sounds like an attendee, attribute it to that attendee - e.g. "Rajasi", "Rajas V" or "Roger V" are almost certainly Rajasvi; "Mia" is likely Mira. Record every such mapping in NOTES as: heard "X", treated as <name>.
- If a mangled name could plausibly be MORE than one attendee (e.g. it sounds like both Emma and Mira), do NOT guess - put that piece of the transcript in NOTES and let the coach place it.
- A name that cannot plausibly be any attendee (a different instructor, a parent, a child clearly not present) stays out of every child's section - record it in NOTES.
- Never place one child's information in another child's section. If attribution is unclear for any other reason, put it in NOTES instead - the coach will sort it out.
- Use ONLY the transcript. Never invent.

Respond in EXACTLY this format, using every attendee id, in this order, nothing else:
===COMMON===
...
===HOMEWORK===
...
${students.map(st => `===STUDENT:${st.id}===\n...`).join("\n")}
===NOTES===
...

TRANSCRIPT:
${transcript}
```

### 6.3 Group stage 2 — per-child report prompt
```
Write a warm, professional GROUP-lesson progress report for ONE child's parents, from the instructor's confirmed notes below. Other children attended, but this report is only about ${student.name}.

CONTEXT:
- Child's first name: ${student.name}
- Activity or subject: ${student.sport}
- Instructor (sign the report as this, exactly): ${coachLabel(coach)}
- Date: ${date}
- Lesson duration: ${state.duration} minutes
- LANGUAGE: write the ENTIRE report in ${state.reportLang}, including naturally translated section labels (keep the ** bold markers). Keep children's names as they are.

CONFIRMED MATERIAL (use ONLY this - nothing else exists):
[What the whole group worked on]
${common}
[What the instructor noted about ${student.name} specifically]
${note.trim() || "(nothing specific - the instructor had no individual note for ${''}this child today)"}
[Group homework]
${homework || "(none given)"}

RULES:
1. Use ONLY the material above. Never invent drills, praise, or events.
2. NEVER name any other child. If the coach's note mentions another child's name, refer to them generically ("a partner", "a teammate").
3. Warm first-person voice, in vocabulary natural to the activity. Parents are the audience. Under 150 words.
4. If there is no individual note, be honest and graceful: describe the group work, and say you'll have more individual detail next time - do NOT fabricate individual observations.

REPORT SECTIONS (markdown, bold labels):
**What the group worked on:** ...
**What I saw from ${student.name}:** ... (omit this section entirely if there is no individual note)
**Notable moment:** ... (ONLY if the individual note contains one; otherwise omit)
**Homework before next time:** ... (omit if none)

Respond in EXACTLY this format, nothing else:
===REPORT===
<report markdown>
===HOMEWORK===
<homework for this child in one sentence, or blank>
```

### 6.4 Clinic prompt
```
A youth instructor ran a CLINIC or workshop (a larger session - sports clinic, ensemble rehearsal, group review session) and described it in one voice memo. Write ONE shared report that goes to EVERY family - it must read naturally for any child who attended.

CONTEXT: Instructor ${coachLabel(coach)}, ${date}, ${state.duration}-minute session, ${count} families will receive this.
LANGUAGE: write the ENTIRE report in ${state.reportLang}, including naturally translated section labels (keep the ** bold markers).

RULES:
1. Use ONLY the transcript. Never invent.
2. Do NOT name, single out, or evaluate any individual child. If the transcript mentions individuals, fold it into general terms ("several players", "the group") and flag the names in NOTES.
3. Warm first-person voice, plural ("we", "the group"). Under 150 words.

REPORT SECTIONS (markdown, bold labels):
**What we did today:** ...
**What stood out:** ... (about the group as a whole; omit if nothing genuine)
**To practice at home:** ... (omit if none given)

Respond in EXACTLY this format, nothing else:
===REPORT===
<report markdown>
===HOMEWORK===
<the at-home practice in one sentence, or blank>
===NOTES===
<individual names you had to generalize, uncertainties; or None>

TRANSCRIPT:
${transcript}
```

## 7. PDF spec ("coach's scorebook" print design)

US Letter, 0.75" margins. Cream page (#F7F5EF); double rule under a serif
banner (ORG_NAME, Times bold) with red Courier header-right ("PROGRESS
REPORT"); serif title "{Name}'s {activity} lesson"; muted Courier meta line
"{date} - {duration} MIN {TYPE} - WITH {COACH LABEL}"; green Courier section
labels; Helvetica body; Notable moment as a white card with red left spine and
italic serif quote; homework as a green-tinted ledger (two columns with
DONE/NOT DONE stamp for private; single full-width box for group/clinic);
italic signature "- {Coach label}"; footer: "ENTRY No. {n} - RECORDED {date} -
PART OF {NAME}'S PERMANENT DEVELOPMENT RECORD" + reply invitation.
Palette: ink #22331F, green #2F6B4F, red #B23A2A, muted #6B7263, rule #DAD5C6,
tint #EDF3EF. Pilot engine is ASCII-only (base-14 fonts, accents
transliterated); a real app should embed a Unicode font so Devanagari etc.
print (until then the UI warns on non-Latin report languages).

## 8. Delivery

Primary: one-tap server-side send — app builds PDF, posts {to, subject, body,
pdfBase64} to backend; backend emails parents with PDF attached and logs the
send. Sending AUTO-SAVES the entry first ("if the parent got it, it's in the
record"). Subject: `{Name}'s {activity} lesson - {date} ({Coach label})`.
Fallbacks, in order: mailto (recipient UNencoded; fired via anchor click),
native share sheet (mailto silently fails in iOS standalone mode), clipboard
(always pre-copied). Failure detection: if the page doesn't lose focus ~1.3s
after the mailto, assume swallowed and offer the fallback sheet.

## 9. Transcription

Primary: record real audio (MediaRecorder; webm/opus, mp4 on iOS), send to
backend, Whisper (whisper-1) with language hint from spoken-language setting
(ISO-639-1). Consistent across devices; ~$0.006/min. Fallback: browser Web
Speech API with hard-won platform rules — rebuild transcript from the FULL
results list every event (Android re-reports whole sessions); on Android use
non-continuous sessions chained by auto-restart (continuous mode duplicates
finals) plus a tail-dedupe guard; auto-restart until user stops (engines quit
after ~60s/pauses); actionable error messages per error code. Raw audio is
currently discarded; the roadmap asset is saving it (future training dataset).

## 10. Activity events (analytics schema)

login, report_generated, group_notes_generated{count},
group_reports_generated{count}, clinic_report_generated{count}, report_saved,
pdf_downloaded, email_sent{to}, transcribed{chars}, webhook_test_app.
Common fields: at (ISO), coach, student?, date?. The email_sent stream is the
truthful "lessons logged" metric; transcribed carries character counts only —
transcript content NEVER reaches the admin log.

## 11. Pilot-era components → real-app replacements

| Pilot (throwaway)                          | Real app                        |
|--------------------------------------------|---------------------------------|
| Sealed AES-GCM roster blobs, passcode login | Real auth (e.g. Supabase/Firebase), server-side roster |
| localStorage per device                     | Database; records visible to admin; survive phone changes |
| Google Apps Script webhook                  | API routes / serverless functions |
| Anthropic key in sealed blob (spend-capped) | Server-side key, proxied        |
| MailApp / mailto chain                      | Transactional email (Resend/SendGrid), coach reply-to |
| Hand-rolled ASCII PDF writer                | Real PDF lib + embedded Unicode font, same layout spec |
| Single HTML file                            | Modules mirroring this doc's sections |

What ports unchanged: §6 prompts, §3 data model, §4 formats, §5 flows and
review gates, §7 layout spec, §10 event schema, §1 principles.

## 12. Field lessons (paid for in debugging; do not relearn)

- JSON is the wrong envelope for content that quotes children; sentinels won.
- Android speech: duplicates finals in continuous mode; whole-session
  re-reports; fix = rebuild-not-append + non-continuous + restart chaining.
- iOS: standalone (home-screen) mode silently swallows mailto; Mail "sends"
  into a dead Outbox when no account is configured; dictation languages are
  device-dependent (Marathi often absent) — cloud transcription removes this class.
- Apps Script: /dev URLs are private, /exec is public; edits are invisible
  until a NEW DEPLOYMENT VERSION; access must be "Anyone" for anonymous posts.
- Every generic name ("Rajasvi" → "Roger V") must fuzzy-match against the
  closed attendee set, with the mapping disclosed, never silent.
- Uniform AI praise across a group is how parents smell the robot; the
  unmentioned-child flag and omitted-moment rules are retention features.
- GitHub Pages caches ~10 min; ?v=N cache-busts; home-screen icons cling longer.
