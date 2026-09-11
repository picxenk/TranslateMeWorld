# TranslateMeWorld

This document defines the requirements and implementation architecture for the replacement app, not a record of verified behavior.

## User Requirements

- A minimal world-map interface for translating Korean or English input into the 56 app target languages listed below, accessible through 78 country markers. App target coverage is not a claim of official model support or verified translation quality.
- Source selection: automatic Hangul detection (Korean when Hangul is present, English otherwise), or explicit Korean/English selection. This is a two-language heuristic, not general language detection.
- Translation starts only with the Translate button or Ctrl/Cmd + Enter, never while typing. Empty input sends no requests.
- Translate targets sequentially and add map links as each result becomes available; do not wait for the whole batch. The source language does not need a translation request. Countries sharing a language share its result without duplicate API calls; translations do not adapt to regional dialects or expressions.
- Group languages sharing a country, especially India's multiple languages, South Africa's Afrikaans/Zulu, and Nigeria's Hausa/Yoruba/Igbo, so every language remains individually accessible.
- Provide world, North America (including Central America and the Caribbean), South America, Europe, Asia, and Africa map navigation for dense regions and small mobile screens rather than relying on tiny overlapping world-map targets. Grow the map canvas height as needed to fit markers; retain horizontal scrolling on narrow screens.
- Selecting a result opens a popup with its translation and an action to save that individual result. Saved entries include source text, translated text, explicit source/target languages, and the chosen country.
- Keep saves in localStorage across reloads, deduplicate identical translations across countries while preserving the first saved country, and allow deletion. Normalize older entries without a country to the language's original representative country on load. Do not automatically save every generated translation.
- New input or a changed source selection cancels the active request and remaining queue. A new translation run supersedes the previous run; stale responses must not update the current map.
- Show connection/model status, translation progress, per-language failures, and retry actions. Preserve successful results when another target fails; retries must use the current source context.
- Offer Listen/Stop for completed translations in both map and saved-history popups. Stop previous speech when another translation is played, the popup closes (including Escape/backdrop), input changes, a new translation run starts, or the page is left. Unrelated translation updates and saves must not interrupt playback.

## Language Scope

Use these 56 unique app target codes: the original 52 in their existing order, followed by four experimental additions (`am`, `ha`, `yo`, `ig`):

```text
ko en ja zh fr de es it pt ru hi ar bn id vi th tr fa he uk pl nl cs sk sl hr sr bg ro hu el da sv no fi is et lv lt ms fil ta te kn ml mr ur gu pa af sw zu am ha yo ig
```

The [official TranslateGemma model card](https://huggingface.co/google/translategemma-4b-it) states 55 languages, but its exact language list has not been verified against this app's list. Do not describe all 56 app targets as officially supported or quality-tested. Amharic, Hausa, Yoruba, and Igbo carry `experimental: true`; map and history popups warn that the model may respond in another language and offer an explicit Google source-translation link for comparison. Nonempty model output is not proof of correct language or translation quality.

Country anchors organize language results; they do not imply that a language belongs exclusively to one country. The map is a simplified illustration, not an authoritative depiction of borders, sovereignty, or language distribution. Current coverage includes 78 country markers in total, with these regional mappings (country codes followed by language codes):

- North America (`northAmerica`), 14 markers including Central America and the Caribbean: US/BZ `en es`; CA `en fr`; MX/GT/HN/SV/NI/CR/PA/CU/DO `es`; JM `en`; HT `fr`.
- South America (`southAmerica`), 12 markers: BR `pt`; AR/CL/PE/CO/VE/EC/BO/PY/UY `es`; GY `en`; SR `nl`.
- Africa (`africa`), 10 markers: existing ZA `af zu` and KE `sw`; added ET `am`, TZ/UG/CD `sw`, NG `ha yo ig`, NE/GH `ha`, BJ `yo`.

See the Korean coverage tables in [README.md](README.md) for the six requested African languages and major Americas languages. These mappings are not exhaustive: French at Haiti's marker is not a claim that it is the only or main mother tongue; Haitian Creole and other Americas creoles and indigenous languages are not currently enumerated as separate targets.

## Implementation Architecture

- One self-contained `index.html` containing HTML, CSS, vanilla JavaScript, and map data/graphics. No CDN, external frontend dependencies, build step, or application backend.
- The browser calls local Ollama directly at `http://localhost:11434`. A local static HTTP server only serves the page; it does not process translations.
- `GET /api/tags` checks connectivity and installed models. Prefer `translategemma:4b`; if absent, use an installed `translategemma` family model. If none exists, show `ollama pull translategemma:4b`. Display the selected model.
- Send one `POST /api/chat` request per unique target language other than the source, sequentially, with `stream: false`, not one request per country marker. Supply explicit source/target language names and codes and request translation only; read `message.content`. No country-specific dialect instruction is sent. Incremental UI updates occur between completed language requests, not token streams.
- Keep language metadata (code, labels, original representative country, experimental flag) separate from country metadata (code, label, region, map position, associated languages). Keep the active source snapshot, results/status keyed by language code, and saved entries in client-side state. Derive displayed language/country counts and progress totals from `LANGUAGES` and `COUNTRIES`, not hardcoded totals.
- Regional tabs use `northAmerica` and `southAmerica` alongside `world`, `europe`, `asia`, and `africa`. Calculate the map's minimum height from visible country count and available marker columns so the responsive grid has enough slots; redraw on resize.
- Use `AbortController` to cancel browser requests and a run identifier to reject late responses. Stop scheduling old targets after cancellation; browser cancellation does not guarantee immediate termination of model computation.
- Render input/model output as text, not executable HTML. Persist only deliberately saved records to localStorage; handle unavailable storage or invalid stored data without breaking translation. Save the selected country as `country`; on load, normalize missing or invalid country-language associations to the language's original representative country. Deduplicate by source text, source language, target language, and translated text, excluding country so the first saved country is preserved.
- Use browser `speechSynthesis` and `SpeechSynthesisUtterance`, with an explicit matching voice whose `localService` is `true`. Prefer the selected country's locale (the persisted country for history), then another locale of the same language; the original representative country is the default when no country is selected. This affects speech only, not the shared translation text. Handle regional/legacy tags (including Norwegian `nb`, Filipino `tl`, and Hebrew `iw`). Never fall back to remote or unrelated-language voices. Refresh availability on `voiceschanged`; disable unavailable playback with a visible explanation. Handle playback errors and ignore late callbacks from canceled utterances. TTS availability and quality depend on the browser and installed OS voices, independently of model translation support; local speech is not guaranteed for all 56 app targets.
- Completed map/history translations also offer an explicit Google Translate link for each language, including the additions, independent of local voice availability. Country expansion leaves this behavior unchanged. Use URLSearchParams to encode the translated text and its language as the input language (`fil` maps to Google's `tl`), open with `target="_blank"` and `rel="noopener noreferrer"`, and stop local speech on activation. Explain before activation that the text is sent to Google in the URL; no Google requests are made automatically. Users must click Google's input-side speaker themselves; speech availability is not guaranteed for every language.
- Experimental languages additionally offer a source-translation link whenever a source snapshot exists, including pending/failed results. Encode the original text, its language as `sl`, and the selected target as `tl`. This explicitly sends the original text to Google on activation; it does not automatically fetch or import a translation. Use the same new-tab security and local-speech cancellation as the result link.

## Connection and Privacy

- Distinguish missing models and API errors when responses permit. A browser fetch failure alone cannot reliably distinguish an unreachable server from CORS blocking; offer guidance without claiming a definitive diagnosis.
- Recommend `http://localhost:8000` instead of `file://`. Check that Ollama is running and the model is installed, then inspect browser errors for CORS or mixed-content blocking.
- For CORS, allow the actual page origin with `OLLAMA_ORIGINS` and restart Ollama. For the documented setup use `http://localhost:8000`, not a wildcard. macOS app users can use `launchctl setenv OLLAMA_ORIGINS "http://localhost:8000"`; Windows users can set that user environment variable before restarting Ollama. Terminal-launched servers must inherit the variable.
- With the default local endpoint, translation text goes to local Ollama, not a cloud translation service. Saved records remain in the browser profile for the page's origin, are not encrypted by the app, and are not account-synced. Changing host/port changes the storage origin; clearing site data removes saves. See `README.md` for setup and usage.
