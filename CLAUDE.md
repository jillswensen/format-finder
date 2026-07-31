CFF (Content Format Finder / Format Finder)
What this is

A Claude-powered content strategy tool built for Bra Fittings by Court (BFBC), a bra fitting boutique in Kaysville, UT. It takes a raw content idea and walks it through format selection, hook generation, full script writing, shot planning, captioning, and production-ready export — replacing what used to be a manual brainstorm-to-Notion workflow.

Live at jillswensen.github.io/format-finder (password-gated)
Single-file app: everything lives in one index.html (HTML + CSS + vanilla JS, no build step, no framework)
GitHub repo: jillswensen/format-finder
Calls the Anthropic API directly from the browser (api.anthropic.com/v1/messages) using an API key stored in localStorage, model claude-sonnet-4-6
Also calls OpenAI Whisper directly from the browser for video transcription (separate API key, also in localStorage)
All user data (businesses, conversations, ideas, scripts, shot plans) lives in localStorage, synced across devices via a Cloudflare Worker + KV store (migrated off an earlier Supabase setup)
Why it exists

Jill runs content for BFBC largely solo. CFF exists to compress "I have a vague idea" → "here's a filmable, on-brand script with shot list and captions" into one tool, using a large curated system prompt (56 short-form content formats, each with mechanism, structure, and BFBC-specific hook examples) instead of relying on ad hoc prompting each time.

Core architecture
CHAT_SYSTEM_PREFIX — the lean, always-loaded system prompt: how the assistant should reason (core message → 2–3 format options with hooks → build full plan once one is picked), script quality rules, caption rules, CapCut editing conventions, brand voice rules.
FORMAT_FRAMEWORKS — a large JS object of the 56 format frameworks (Common Belief Buster, Dry Parody, How to 5 List, Blind Ranking, One Peak Style Ad, etc.), each with mechanism/structure/BFBC-specific examples. Kept separate from the system prompt; only the framework matching the idea's chosen format is injected in per-idea tabs via getTabSystemPrompt(), to keep token usage down.
Multiple businesses supported (multi-tenant by design), though BFBC is the only one actively used.
Ideas are the central object: each has a hook, a chosen format, notes, an optional transcript, a script chat history, a caption chat history, and — once generated — a shotPlan (the Beat View).
Tabs / features
Brainstorm — open-ended chat; core message → 2–3 format cards with hooks (returned as a fenced JSON block per the system prompt, parsed into UI cards) → pick a direction.
Analyze — paste/upload a posted reel's transcript + metrics (or screenshot), get a performance diagnosis grounded in account-specific benchmark data hardcoded in ANALYZE_SYSTEM (skip rate, saves, before/after benchmarks, etc.).
Ideas — the pipeline/kanban view of all saved ideas, status-tracked (Idea → Scripting → Ready to Film → Posted → Analyzed).
Idea detail has five sub-tabs: Notes, Transcript, Script, Caption, Beat View (labeled "Filming" internally).
Script tab: generates a full script (freeform Claude chat output, not structured JSON) with four sections — Script (A-roll/B-roll), On-Screen Text, Cover Image, CapCut Editing Notes. "Use Notes as Script" lets Jill paste an already-finished script instead of generating one.
Beat View / Filming tab: takes the generated script and asks Claude to restructure it into a shotPlan — a structured JSON array of beats, each with title, script (verbatim spoken line, never reworded), visual, alt, textOverlay, notes[]. This is the only structured (non-freeform) representation of a script anywhere in the app, and is the backbone for exports.
Export for Notion modal (modalExportScript, added July 2026): three formats, all built exclusively from shotPlan (no fallback — requires Beat View to be generated first):
Full Script — the raw generated script, markdown stripped for clean pasting.
Talent Sheet — Jill's fixed handout template (Role/Outfit/Props/Notes as fill-in-yourself placeholders; "Your Shots" auto-filled from each beat's visual; "Your Lines" auto-filled from each beat's script). No per-talent splitting logic — one shared sheet per idea, by design.
On-Screen Only — each beat's textOverlay, labeled by clip number (array position, not any stored ID).
Caption tab: 3 caption options (Option A/B/C) following brand caption rules, using the locked script as context when one exists.
Stories — separate chat tool for Instagram Stories sequences (different system prompt, frame-by-frame output with interactive sticker suggestions).
Fitting — upload .txt/.vtt fitting-session transcripts → Claude finds before/after moments and generates 2–3 reel concepts using only existing footage (every clip reference cites filename + timestamp; fitter names are never used in hooks/on-screen text). Concepts are parsed into fittingConceptStore (in-memory JS, not HTML attributes, to avoid truncation).
Data model notes (important for future edits)
Nothing in the data model tracks talent identity per clip/line — scripts and beats are talent-agnostic by design.
Beat title fields are human-readable labels ("THE HOOK", "THE REVEAL"), not clip IDs — anything that numbers clips does so by array position.
Script generation output is unstructured markdown text (a single assistant chat message), not JSON — the Beat View step is what turns it into structured data. Any future feature that needs structured script data should hook off shotPlan, not try to parse the raw script text.
All persistence is localStorage + Cloudflare KV sync — there is no backend database beyond that KV store.
Conventions to preserve when editing
Single-file architecture — no build step, no bundler. Keep it that way unless explicitly asked to restructure.
New pill-style toggle buttons (format switchers, filters, tabs) should get real CSS classes with an explicit appearance:none; -webkit-appearance:none; outline:none; reset rather than relying on inline styles per button — inline-only styling on these has caused native-button chrome artifacts before.
Script content is precious: any AI step that transforms an existing script (e.g. Beat View generation) must copy spoken lines verbatim rather than paraphrasing — this is explicit in the Beat View system prompt and should stay that way in any related feature.
Brand rules that must never be silently dropped in prompt edits: no fitter/talent names in hooks or on-screen text; no local/geographic hashtags unless the content is explicitly about the Kaysville location; captions always offer both CTA paths (in-person fitting + free online size calculator); Before/After reels are exempt from the ≤30s duration guidance, everything else isn't.
Deploys are manual: changes get uploaded via GitHub's "Upload Files" method (the file's too large for the inline web editor), ~2–3 min to go live.