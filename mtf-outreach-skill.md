# MTF school outreach skill

Outreach to school-level Chinese Language / Mother Tongue decision-makers (HODs, Subject Heads, Lead Teachers, Principals) for [Product], an interactive Mandarin mystery-solving game for Chinese Mother Tongue Fortnight.

This skill is the single source of truth for how the MTF workflow is done. Scheduled tasks and chat sessions both follow it. A task prompt decides *which* population and *when*; this skill decides *how*. Where they disagree on how, this skill wins.

The user works in the CRM as **[SDR name]** and signs the emails as [SDR name].

## Tool routing

The team's real working data lives in a workspace/CRM tool, not general email, so mixing these up reads the wrong system entirely:

- ALL CRM database reads/writes go through the CRM connector tools. Never the email connector for CRM data.
- The email connector is scoped to MTF email drafting and sending only.
- CRM tools are usually deferred: load them before use, batching everything you expect to need into one call.

**Base identifiers.** Pin the exact app token and the Accounts / Contacts / Schools table IDs and field IDs here, so every read and write targets the correct field without guessing. Keep them in this section only; never hardcode an ID anywhere else or recall one from memory mid-session.

## 0. Never create, edit, or delete a field's options

The CRM tool silently auto-creates a new option on a single-select or multi-select field the instant you write a string that isn't an exact match to an existing one. There is no error, no confirmation, and no warning — the write just succeeds, the field schema quietly grows a new option, and every other record's dropdown now has junk in it. This has actually happened (an invented value on the research-status field, and unsuffixed variants of two multi-select options that didn't match the real option strings, one of which has a trailing space) — treat it as a live failure mode, not a hypothetical one.

The only actions ever permitted on a select/multi-select field are: read the field's current option list, then write one or more of the exact, already-existing option strings back. Never invent a new value, never "clean up" or rename an option, never delete one, and never assume you know the valid set from this document's prose — the prose can drift from the live schema (exactly what happened above: nothing ever said what value to write, so a value got invented).

Concretely, before the first write to any select/multi-select field in a session:
- Read the field's exact `options` array for that field (name and, ideally, id) directly from the CRM schema.
- Match by exact string. A short label is not the same as its full parenthetical form. Trailing spaces are part of the string — some real options have one, distinct from the version without it.
- The research-status field's real options are `Not Researched`, `Strong`, `Usable`, `None` — the field IS the classification, not a generic "have I researched this" flag. Once section 3's classification is decided, write that exact classification word into that field. There is no separate "Researched" state to set.
- If the value you need to write isn't among the existing options, STOP. Do not write a close approximation and do not add the option yourself. Flag it to the user with the field name, the value you wanted, and the current option list, and let them add it (or tell you which existing option it maps to).

## 1. Account eligibility gate — check before touching ANY record

The Accounts table is **shared with other salespeople**. This is the reason the gate exists: writing to an account that is not yours can destroy a colleague's live pipeline. Flipping their in-progress-deal stage to "First Email Drafted" erases the real stage, they miss a follow-up, and a client is lost. That damage is invisible until it is too late, which is why the check has to happen before the work, not after.

**Stage alone is never sufficient for eligibility. All four of the following must be true, verified on the record itself immediately before any write:**

1. **`Owner` is exactly your name.** `Owner` is a **MultiSelect**, not single-select. *Containing* your name is not enough — your name plus any other name means the account is co-owned with another salesperson and is out of scope. Skip it and flag it by name. A co-owned account looks safe and is not.
2. **`Type` = `New MTF Account`.** Other account types (existing accounts, past customers, stalled/lost) are out of scope.
3. **The linked contact's "MTF Contact" checkbox is checked.** The source field lives on the **Contacts** table, but Accounts carries a live lookup of it through the linked Contact record — include it directly in the initial eligibility query rather than reading each candidate's Contacts record separately. An empty Contact link, or an unchecked lookup value, fails the gate.
4. **`Stage` is exactly what the action expects** — one status for research/drafting, a later status for sending — re-read immediately before the write. Querying at the start and writing minutes later is how a record that changed underneath you gets clobbered.

**Stage movement.** The ladder holds real sales stages (Qualified, Meeting Held, Opportunity, Proposal/Tender, Won Deal, Delivered, Pending Client, and similar) that colleagues rely on. Only two transitions are ever permitted: "Account Created" → "First Email Drafted", and "First Email Drafted" → "Contact Established". Never move a Stage backwards or sideways, and never "correct" a stage that looks wrong — it probably reflects something a human knows and you don't.

**When in doubt, fail closed.** If ownership can't be determined, the contact link can't be resolved, or the schema doesn't match this description, stop and report rather than proceed on assumption. Skipping a legitimate account costs a day; writing to a colleague's account can cost a client. That asymmetry is the whole argument. Never write to another salesperson's record for any reason, including to undo an earlier mistake — surface it and let the user coordinate with the owner.

## 2. Research stage 1 — Chinese Focus Signal

Run this **before** MTF activity research on every school, because what it finds changes how hard the next stage has to dig.

**Before researching anything, check whether this school has already been done.** On the Schools table: if the Chinese Focus Signal field is not "Not Checked", reuse the stored value and its detail field instead of re-deriving them. Separately, if the research-status field is not "Not Researched", skip section 3's MTF activity research entirely and reuse the existing personalisation hook, research source, and research notes for this account's draft. This is the same reuse pattern as School Short Name in section 5 — a school is researched once, no matter how many contacts at that school later become eligible accounts.

Recorded on the Schools table: a Chinese Focus Signal (multi-select) field and a matching detail (text: per-signal detail + source URL) field.

**Capture every signal that applies.** There is no cascade and no stopping early — the point is accurate scoring later, and a school that is SAP *and* clan-affiliated *and* bicultural is a different prospect from one that is only SAP. Check each independently:

1. **SAP** — on the MOE SAP list / self-identifies SAP.
2. **CLEP** — Chinese Language Elective Programme (a small fixed list of Secondary schools; verify against the current MOE list rather than assuming).
3. **Bicultural (BSP/BiCEP)** — BSP (secondary) or BiCEP (primary); a small fixed list of primaries runs BiCEP.
4. **Clan-affiliated** — Chinese clan/association link (e.g. a major Chinese clan association). Record it even if a stronger signal already fired; it is additive, not a fallback.
5. **Chinese Flagship** — an obvious Chinese-focused flagship programme not caught above. The only one needing a per-school page check; the other four are cheap list lookups and should be checked regardless.
6. **None** — only after checking all of the above.

No weights or tiering; all signals are equal for now. Any scoring formula gets derived later from real closed-won outcomes rather than guessed up front.

**Any single non-None signal escalates that school's MTF research to the extra bar in section 4.**

## 3. Research stage 2 — MTF activity protocol

Classification quality depends on search *depth*, not on how fast the first result comes back. (If the research-status field was already set, per the check in section 2, this section is skipped entirely.) Record what was checked in the research-source / research-notes fields for every classification, "None" included — an unrecorded classification can't be audited, so a wrong one can't be caught.

MTF content rarely lives on the homepage or top nav — it typically sits on a department subpage that a shallow search misses entirely, which is why depth matters more than speed here.

1. **The school's own official website first.** Aggregator/directory sites carry no MTF detail and must never be the basis for a classification — use them only to locate the official domain.
2. **Open the Mother Tongue / Chinese department subpage directly.** Try common department-page paths. If a URL 404s, fall back to site-scoped searches for the fortnight by name, and in Chinese as well as English.
3. **The school's own social channels.** Search for the school name plus "mother tongue fortnight" on Instagram/Facebook. Many schools publicise MTF there far more than on their site. Also newsletters and yearbooks (often PDFs).
4. **Search in Chinese, not only English.** English-only search misses a lot, especially at Chinese-strong schools.
5. **Only after 1-4 come up genuinely empty** may "None" be considered, and only with a record of exactly what was checked.

**Efficiency note, with a hard boundary.** A search snippet can be used to skip fetching the full page, but only when the snippet itself already names a describable MTF-tied activity sufficient for the hook. An empty, vague, or unrelated-looking snippet is never grounds to skip a fetch or shorten the protocol — absence in a snippet is not absence on the page. The site-scoped fallback queries may be shortened to the first query returning a clear, describable hit, but only for schools carrying no Chinese Focus Signal from section 2. Any signal-carrying school still runs the full protocol including the Chinese-language queries, regardless of what an early English hit finds — section 4's extra bar already requires full exhaustion for these schools, and stopping early here would undercut that.

## 4. SAP and signal-carrying schools — the extra bar

SAP schools exist specifically to promote bilingualism and Chinese language and culture. They run MTF essentially by definition, usually elaborately. So a SAP "None" is almost certainly a research failure rather than a true negative, and should be treated as evidence that the search was not deep enough. Getting a neighbourhood school wrong is recoverable; getting a SAP school wrong is not, because they are the top-priority ideal client profile.

The same bar applies to any school carrying any Chinese Focus Signal.

- Exhaust the full protocol, then also check the BSP page, SAP / 华文 flagship programme pages, learning-journey writeups, immersion-trip and camp pages, and competition results.
- **If still nothing after all that, do not quietly write "None."** Flag it with the exact list of pages and channels checked and ask. In a scheduled run: leave Stage unchanged, skip drafting, and flag it in the summary so a human can look.

**Known SAP / CLEP / BiCEP / clan-affiliated school lists.** Maintain the current, name-verified lists here: SAP secondary and primary schools, CLEP secondary schools, bicultural-programme primaries, and clan-affiliated schools. Also list the schools that look Chinese-focused but are confirmed not SAP, so they are not miscategorised on name alone. Re-verify the lists periodically against the official MOE list rather than trusting this file's copy indefinitely.

## 5. School short name — mechanical rule, no research, no abbreviations

**Short names are always derived mechanically, with the sole exception of Junior Colleges (JC), whose short names have already been pre-filled on the Schools table. For primary and secondary schools, no abbreviations or aliases are used, ever, regardless of how strong the first-party evidence for one might be.** Do not go looking for a school's own acronym or handle for this purpose.

Rule: drop the trailing generic institutional noun from the school's full name (School, Institution, Institute, Academy, and similar), and use the remainder in sentence case. E.g. "Riverbend Primary School" → "Riverbend Primary"; "Westvale Junior College" → "WJC" (JC exception). Never reproduce scraped ALL CAPS.

Check the short-name field on the Schools table first: if it already holds a value produced by this mechanical rule, reuse it. If the record is not a "JC" or "Mixed" school type, and its short-name field holds an abbreviation instead, recompute the mechanical short name and overwrite it. Write the result back so future drafts don't need to recompute it.

This is a deliberate simplicity-over-personalization tradeoff. Don't reintroduce abbreviation-hunting even if a strong self-reference signal surfaces during research.

## 6. Classification and hooks

An activity only counts as evidence for "Strong" or "Usable", and may only appear in the hook, if the source explicitly ties it to Mother Tongue Fortnight / MTL Fortnight / MTF by name. A general Chinese department page, an occasion-specific event (Chinese New Year, Mid-Autumn, Hari Raya), an ongoing curriculum initiative, or something the source lists as running "alongside" MTF does not qualify on its own, however active and well-documented it is. Writing a hook that implies otherwise is the kind of small inaccuracy a HOD notices immediately, and it costs the credibility of the whole email.

**Do not name Mother Tongue Fortnight twice in the same sentence, under any of its names.** MTF, Mother Tongue Fortnight, MTL Fortnight and MT Fortnight are the same thing wearing different clothes, and the hook sentence already establishes which one we're talking about ("has been running MTF activities such as..."). Once it's named, everything after "such as" describes *what the school did*, not *when or why* — so the activity description should never smuggle the event's name back in. This is a real, recurring failure, not a hypothetical one:

> ❌ "[School] has been running MTF activities such as card making, tea appreciation and folk dance over a two-week Mother Tongue Fortnight."

That sentence says MTF twice: once directly, once as "a two-week Mother Tongue Fortnight" tacked onto the end. The second mention adds nothing a reader doesn't already have, and it reads as if the sentence forgot what it already said three words earlier. The fix is to end the sentence once the activities are listed:

> ✅ "[School] has been running MTF activities such as card making, tea appreciation and folk dance."

Before finalizing any hook, read the activity clause on its own and ask: does it name the fortnight again, under any spelling? If a research note reads something like "held during their two-week Mother Tongue Fortnight" or "as part of MTL Fortnight," that phrase describes *when*, which the sentence has already covered by using "MTF" up front — drop it rather than transplanting it into the hook. The same logic applies to the "Strong" and "None" hooks and to the fixed "Chinese MTF" phrase in the master email itself: MTF is named exactly once per email, in the subject line and the opening paragraph, and never redundantly restated elsewhere in different words.

The school-specific paragraph is four sentences maximum, problem → solution → benefit, and never invents a school fact.

**Strong** — clear recent evidence of game-based or gamified learning, interactive Chinese learning, student-led investigation, or creative MTF formats:

> [School short name] has been using [activity/initiative] to make learning more active. [Product] applies the same learning-through-play approach to spoken Mandarin. Students get absorbed in solving the mystery and speak Mandarin because it helps them progress and win, not because they have been asked to practise. We run it from setup to facilitation, so it gives you another ready-to-run way to build on the kind of active learning your school is already investing in.

**Usable** — recent MTF activity, but mainly cultural workshops, performances, calligraphy, craft, festive activities or talks. Never imply these are stale or ineffective; the goal is to sit alongside them:

> [School short name] has been running MTF activities such as [activity], [inferred outcome]. [Product] could add a new format to your existing programmes by promoting language use through game-based learning. Students get absorbed in solving the mystery and actively speak Mandarin because it helps them progress and win, not because they have been asked to practise. We run it from setup to facilitation, so your MTF programme gets a fresh gamified addition without extra planning work for your teachers.

[activity] is a plain list of what was done ("card making, tea appreciation and folk dance"), followed by a comma and [inferred outcome]: a short clause naming what that specific documented activity already achieves for the school (e.g. "getting students performing and having fun in Mandarin" for a karaoke competition, "building appreciation for Chinese culture and heritage" for a calligraphy workshop). This is inferred per school from the research notes, not a fixed phrase, because Usable-tier activities vary too widely — from near-Strong-caliber interactive formats to purely passive exposure — for one generic outcome to fit all of them. It must never invent detail beyond what research documented; if the research notes already state an explicit goal for the activity, paraphrase that instead of inferring fresh. Record the basis for [inferred outcome] in the research-notes field alongside the activity, so it is reviewable during the approval pass rather than invisible inside the draft body.

Neither [activity] nor [inferred outcome] may continue into a time frame or a restatement of the event name — no "over a two-week Mother Tongue Fortnight", no "during MTL Fortnight", no "for MT Fortnight". If the research notes describe *when* the activities happened, that context has already been served by "MTF" earlier in the sentence and gets left out here. The "name MTF once per email" rule above applies to [inferred outcome] too — it describes what the activity achieved, never re-names the fortnight.

**None** — a positive claim that the school does not visibly run MTF, made only after the full protocol. It is not a default for "I didn't find it quickly." Before using it: the school's own MTL/Chinese subpage was opened, what was checked is recorded, and the school is neither SAP nor carrying any Chinese Focus Signal (those escalate instead):

> Getting students to speak Mandarin willingly outside normal classroom practice can be difficult. [Product] turns the language into part of the game. Students get absorbed in solving the mystery and speak Mandarin because it helps them progress and win, not because they have been asked to practise. We run it from setup to facilitation, so your teachers get a ready-to-run programme for MTF without additional planning work.

*Borderline (MTF named but no describable activity):* the "Usable" hook needs a real "such as [activity]", so without one, use the "None" hook rather than inventing detail, and note "borderline — MTF confirmed, activity detail thin" in research notes so it can be revisited.

The third sentence ("Students get absorbed...") is fixed copy reproduced word for word in all three branches, as is the "We run it from setup to facilitation" sentence. These carry the core pitch and were settled deliberately; do not reword or vary them per school.

## 7. The master email

**Subject:** [Campaign name] at [School short name]

> Dear [Honorific] [LastName],
>
> I'm [SDR name] from [Company name]. For the past four years we have run gamified learning programmes with over 100 schools in [Country], reaching more than 15,000 students.
>
> We are developing [Product], an interactive Mandarin mystery-solving game for Chinese MTF, and we would like [School short name] to be among the first schools to try it.
>
> [SCHOOL-SPECIFIC PARAGRAPH]
>
> Would you be open to a short meeting at [School short name] so I can share more and see whether it fits what you are planning? We would also be happy to run a demo for you and your teachers first.
>
> You can take a quick look here: [product URL]
>
> Best regards,
> [SDR name]
> Programme Lead, [Company name]

Do not rewrite the fixed parts unless explicitly instructed.

**Voice and style**, applying to every piece of drafted copy. The aim is that it reads like one person writing to another, so anything that smells of a template or of an LLM undercuts it:

- No time-bound greeting ("Good morning"). Drafts get sent and read at all hours.
- No em dashes anywhere. Use a comma, period or semicolon.
- No AI-sounding filler: "in today's fast-paced world", "unlock", "elevate", "seamless", "delve", "furthermore", "moreover", "boost engagement".
- No contractions, with one exception: "I'm" in the opening line.
- Do not open the school-specific paragraph with "We saw that", "I noticed that", "I came across". Start with the school as the subject, or with the problem statement in the None branch.
- No redundant restatement of the same idea in different words within one sentence or paragraph, MTF/Mother Tongue Fortnight naming being the specific known case (see section 6), but the same instinct applies generally: say a thing once and move on.
- Warm, direct, professional. Never criticise the school's existing programme. Do not over-explain game mechanics.

**Salutation.** "Dear [Honorific] [Last Name]," using the Honorific field on Contacts. There is no Last Name field — infer the surname from Contact Name plus the email local part, since local convention often means an English alias replaces a given name while the surname stays recognisable in the email address. For patronymic-style names (given name plus father's name, with no family surname) address by given name instead. Best-effort it without flagging; the user corrects during review.

## 8. Email drafting mechanics

- Set the CTA as an explicit HTML anchor with the URL as visible text, so link-safety rewriting by the mail provider doesn't turn it into an ugly redirect for the recipient. The wrapped link still resolves correctly.
- **Never edit a draft using a Draft ID recalled from memory or from earlier in the conversation.** Look it up fresh from the account's draft-ID field immediately before the call and confirm which contact it belongs to. IDs created in the same batch look alike and are easy to transpose.
- **After any update or create of a draft, re-fetch that same draft** and check the returned recipient, subject and body match the intended contact before reporting it done. A success response only confirms the write landed somewhere, not that it landed on the right draft.
- **Before touching an existing draft (e.g. a bug fix pass), check whether the user already hand-edited it.** Compare against what the fixed template copy and this skill's rules would have produced. If the draft's content deviates from what the rules imply in a way that looks like a deliberate edit (not just a research-specific fill-in), leave it alone and flag it rather than overwriting a human's change.

## 9. Workflow and CRM stages

1. **Add accounts.** Created with account type "New MTF Account", Stage "Account Created".
2. **"Generate drafts"** — pull Stage "Account Created", then apply the eligibility gate in section 1 to every candidate. Stage alone never defines the population. Anything failing the gate is skipped and listed, never processed.
3. **Per account:** Chinese Focus Signal → MTF research → classify → write the research fields → derive short name (mechanical rule, section 5) → draft the email. Write all Schools-table research fields in a single update call per school rather than one call per field. Then flip Stage to "First Email Drafted" and write the draft ID in the same update. Research at the school level rather than per contact, and reuse the approved paragraph across contacts at the same school.
4. **The user vets drafts** and checks a Draft Approved flag themselves, over as many sittings as they need. Never set that flag on their behalf — it is the human sign-off the whole send step depends on.
5. **Send** — only accounts with Stage "First Email Drafted", Draft Approved checked, and passing the full gate. Confirm the draft's recipient matches the account's linked contact before sending. After a successful send, flip Stage to "Contact Established".

**Sending.** Never send automatically from a chat session without the user's explicit go-ahead for that specific round. The one standing exception is a fixed daily scheduled send task, where Draft Approved on each account is that account's sign-off. Do not extend that exception to anything else.

## 10. Reporting

Every batch run reports: how many accounts passed the gate and were processed; the Strong/Usable/None breakdown; which schools carried a Chinese Focus Signal; which short names were derived; schools flagged for manual escalation; and every record skipped by the gate and why, especially anything co-owned or owned by another salesperson. A silent skip hides exactly the information the gate exists to produce.
