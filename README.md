# Sookjai Health (สุขใจ เฮลท์): hackathon MVP

Sookjai Health is a single-page Thai web app for basic health guidance, designed for older adults. Its mascot is **น้องสุขใจ**, a chipmunk drawn from `Character_Design.jpg`. The app has five parts: a camera pulse estimate (PPG), a rule-based symptom checker, a chat with the animated mascot, Thai text-to-speech, and emergency information (1669).

**Live demo:** https://chaiporn2005.github.io/sookjai-health/ (open it on a phone to use the camera pulse measurement)

> **Prototype, not a medical device.** No part of this app has been clinically validated. Pulse values are estimates. Symptom guidance is general information, not a diagnosis, and has not yet been reviewed by a clinician.

## Files

| File | Purpose |
|---|---|
| `index.html` | The whole app: HTML, CSS and JavaScript in one file. It has no external requests and no dependencies. The mascot is inline SVG, so the image file is not needed at runtime. |
| `Character_Design.jpg` | The character design sheet the theme and mascot are based on (reference only). |
| `ppg_validation_template.csv` | A blank paired-measurement table with Excel/Sheets formulas for absolute error, MAE, bias and the share of readings within ±5 BPM. |
| `README.md` | This file. |

## How to run

**Symptom checker, TTS and demo mode:** double-click `index.html`. It works offline in any modern browser.

**Camera pulse measurement on a phone:** browsers only allow the camera on **HTTPS** or `localhost`. Use one of these:

1. **GitHub Pages:** push `index.html` to a repo and enable Pages. Then open `https://<user>.github.io/<repo>/` on the phone.
2. **Netlify Drop:** drag the folder onto app.netlify.com/drop and open the HTTPS link it gives you.
3. **Laptop only:** run `npx serve .` or `python -m http.server`, then open `http://localhost:…`. A phone that loads the laptop's LAN IP over plain HTTP is *not* a secure context, so the camera will be blocked there.

Android Chrome works best, because it is the only common browser that lets a web page turn on the flash. iPhone Safari runs the measurement with ambient light only.

## What is real, simulated and unvalidated

| Feature | Status | Notes |
|---|---|---|
| Camera PPG pulse estimate | **Functional, unvalidated** | Processes real camera frames on the device and runs signal-quality gates before showing a number. It has not been compared against a reference device on real people. |
| PPG demo mode | **Simulated** | Generates a synthetic pulse waveform (target 62–86 BPM) and runs it through the same algorithm. It is labeled "ข้อมูลจำลอง" (simulated data) everywhere, and its rows are excluded from MAE. |
| Symptom checker | **Functional, not clinically reviewed** | Covers 14 symptoms, 9 red flags and 5 context modifiers (age, pregnancy, chronic disease, severity), plus duration. Uses keyword matching on free text. It does not diagnose or recommend medication. |
| Chat with น้องสุขใจ | **Functional, rule-based** | Keyword intents, with emergency and self-harm checks always run first. Not an LLM, so it only understands phrases it was given. Text input only. |
| Thai TTS | **Functional, device-dependent** | Uses the Web Speech API. It needs a Thai voice installed, and the app shows a status message when none is found. |
| Validation table | **Functional** | Kept in memory only. Computes \|error\| and MAE, and exports to CSV. |
| History, accounts, SpO₂, AI chat | **Not built** | Deliberately out of scope, to keep the privacy risk low. |

## Features

**Theme and mascot (from `Character_Design.jpg`)**
- The palette comes from the sheet: cream paper with a dot texture, chipmunk browns, a cream belly, orange inner ears and peach cheeks. Acorn and leaf doodles echo the sheet's title. Section badges are acorn-shaped.
- น้องสุขใจ is hand-built inline SVG with every pose and expression from the sheet:
  - **Front/Wave:** waves on page load, on a greeting, and when you say goodbye.
  - **Side/Walk:** walks into the chat panel when it opens.
  - **Excited:** hops while holding an acorn after a good result, a thank-you or a compliment.
  - **Happy / Surprised / Curious:** surprised jump at an emergency, head-tilt while "thinking", plus a worried face for urgent results.
  - **Idle:** breathing, tail sway, blinking and ear twitch.
  - **Talking:** the mouth moves in sync with speech.
- Tap the mascot on the home screen for a random friendly line and reaction.
- A head-only version is used in the header and the chat button. Every animation stops under `prefers-reduced-motion`; expressions still change.

**Chat: "คุยกับน้องสุขใจ"**
- A floating button opens a chat panel. Type a message or tap a quick reply (pulse, symptoms, emergency, tips, privacy, who are you).
- Replies show a typing indicator. They are spoken in Thai when a voice exists; otherwise the mouth animates silently. Some replies include action buttons: call 1669 or 1323, go to a section, run the full symptom check on what you typed, start demo mode, or enlarge text.
- **Safety order:**
  1. Self-harm phrases → 1323 and 1669.
  2. Emergency symptoms or red flags → urgent red reply with a 1669 button.
  3. Other symptoms → one self-care hint plus an offer to run the full rule-based check.
  4. Small-talk and how-to intents.
  5. A fallback that mentions 1669.
- The chat never recommends medicine or names a diagnosis. Messages stay in page memory.
- **No voice input on purpose.** Chrome's Web Speech recognition streams audio to Google's servers, which would break the "health data never leaves the device" promise.

**Layout and accessibility**
- One page with five sections, reached through a tab bar: Home, วัดชีพจร (pulse), เช็กอาการ (symptom check), ฉุกเฉิน (emergency), วิธีใช้ (how to use). The tab bar sits at the bottom on phones and in the header on desktop.
- A **1669 call button is always visible** in the header.
- Base text is 20px. A `ก+` button cycles the text size through 100%, 112% and 125% and remembers the choice; this is the only thing stored on the device.
- Touch targets are 48–60px. Focus rings are 4px and visible. There is a skip link and real `<label>`/checkbox controls. Live regions announce status changes, and focus moves to each new result.
- Respects `prefers-reduced-motion`. The mascot blinks, waves, moves its mouth while speaking, and shows a worried face for urgent results.
- Uses only system Thai fonts (Sarabun, Noto Sans Thai, Leelawadee UI, Thonburi, Tahoma). No web fonts are loaded, so the page works offline.

**PPG pipeline (`PPGCore` in `index.html`)**
1. Opens the rear camera at about 320×240 and 30 fps. Turns on the torch where `getCapabilities().torch` is supported. If the first rear lens has no torch, it tries the other rear lenses.
2. Each frame: averages R, G and B over the central 50% crop (downsampled to 48×36), and computes the spatial variation of red.
3. **Finger detection:** red share ≥ 0.45, uniform image (coefficient of variation ≤ 0.35), and red brightness ≥ 40. A finger must be held for 1 s, followed by a 3 s exposure-settling period. If the finger is lost for more than 1.2 s, measurement restarts.
4. Uses the red channel, or green if red is saturated by the flash. Records for 20 s. Counts frames that change by more than 5% as motion.
5. Analysis: resamples to a uniform 30 Hz, inverts, and band-passes 0.7–3.5 Hz (zero-phase biquads) to cover 42–210 BPM. It then estimates the rate two independent ways: **(a) peak detection** with a minimum spacing and a prominence filter, using the median of inter-beat intervals with outliers rejected; **(b) autocorrelation** with harmonic and octave correction.
6. **Quality score (0–100):** combines autocorrelation strength, inter-beat-interval regularity, agreement between the two methods, the share of valid beats, and a motion penalty. A number is **shown only** if the score is ≥ 65 (≥ 80 when below 20 fps), the two methods agree within 15%, and the result is 40–180 BPM. Otherwise the app says "วัดไม่สำเร็จ" (measurement failed) and lists the reasons.

**Symptom checker rules (`SymptomCore`)**
- Results come in four levels: 0 ดูแลตนเองได้ (self-care) · 1 ควรปรึกษาเภสัชกร/แพทย์ (consult a pharmacist or doctor) · 2 ควรพบแพทย์ภายใน 24 ชม. (see a doctor within 24 hours) · 3 **ฉุกเฉิน โทร 1669** (emergency).
- **Always level 3:** chest pain, difficulty breathing, sudden arm or leg weakness, facial droop or slurred speech, loss of consciousness or seizure, cyanosis, sudden facial or throat swelling, vomiting or coughing blood, thunderclap headache, fever with stiff neck. Thoughts of self-harm also give level 3 and add the 1323 mental-health hotline.
- **Escalation rules:**
  - Fever with abdominal pain → 2
  - Fever with rash → 2
  - A vulnerable person (age 60+, under 5, pregnant, or with a chronic disease) who has fever, vomiting or diarrhoea → 2
  - Severe or worsening symptoms → 2
  - Symptoms for more than 2 weeks → 2
  - Fever for more than 3 days → 2
  - Other symptoms for more than 3 days → 1
  - 4 or more symptoms → 1
  - Vomiting with diarrhoea → 1, plus a dehydration note
- Each result explains why it was given ("ทำไมน้องสุขใจแนะนำแบบนี้"). It lists self-care steps, warning signs and when to see a doctor for each chosen symptom.
- **Free-text keyword matching** fills in the checkboxes for the user to confirm. It does basic negation: "ไม่มีไข้" ("no fever") does not tick fever.

**Thai TTS**
- The mascot speaks a greeting and reads symptom and pulse results aloud. Results are split into chunks, because Chrome cuts off long utterances.
- Numbers are spelled digit by digit ("หนึ่ง หก หก เก้า" for 1669).
- A floating "หยุดอ่าน" (stop reading) bar appears while speaking; Esc also stops.
- If the voice list has loaded and has no Thai voice, the app shows a message instead of reading Thai in an English voice.

## Privacy

- No server, no analytics and no network requests. Camera frames are drawn to an in-memory 48×36 canvas, averaged, and discarded.
- Symptoms, pulse results and validation rows live only in page memory and are gone when the tab closes. CSV export is a local download.
- Only the text-size preference is kept in `localStorage`.

## Testing

### Automated checks already run (October 2026)

Algorithm tests ran in Node.js against the code blocks extracted from `index.html`. The browser checks used headless Chrome driven over the DevTools protocol.

| Check | Result |
|---|---|
| JavaScript syntax (`node --check`) | Pass |
| PPG on synthetic camera-like signals: 45–170 BPM × 15/24/30 fps, with frame jitter, drift, noise and HRV | 27/27 within ±3 BPM or correctly refused at 15 fps; MAE 0.69 BPM |
| PPG refuses bad input: pure noise, flat signal, short recording, 6 fps, very noisy | All refused (no number shown) |
| PPG stress test: 600 random signals, signal-to-noise roughly 0.3–15 | Reported 35% (refused the rest). Of those reported: MAE 1.8, p95 3.5 BPM, 3 errors > 5 BPM (all near 160 BPM, a rate unrealistic at rest) |
| Demo mode end-to-end in the browser | Target 74 → computed 74, labeled as simulated, excluded from MAE |
| Symptom rules: 14 scenarios and 6 free-text parsing cases (incl. negation) | All pass |
| Chat brain: 13 cases (emergency beats greeting, self-harm, "อยาก" not read as medicine, "สบายดี" not read as "bye", refusing to recommend medicine, fallback) | All pass (70 checks in total) |
| Chat in the browser: open, send messages, get an urgent reply, Esc closes and returns focus, falls back silently without a Thai voice | Pass; no console errors |
| Camera permission denied | Thai explanation shown, plus retry and demo buttons |
| Camera granted (Chrome fake device) | No finger detected (correct), no torch reported, Stop releases the camera |
| Layout at 320 / 390 / 1366 px, all 3 text sizes, with results shown | No horizontal scroll, no console errors, no visible targets under 44px |

> The synthetic results only show that the **algorithm logic** is sound. They say **nothing** about accuracy on real fingers and real cameras. See the validation protocol below.

### Manual test checklist

| # | Area | Steps | Expected |
|---|---|---|---|
| 1 | Mobile layout | Open on a phone (360–430px wide); scroll all sections; try `ก+` three times | Bottom tab bar, no sideways scrolling, text grows, 1669 always visible |
| 2 | Desktop layout | Open at ≥1000px | Tabs move into the header; the hero shows two columns |
| 3 | Navigation | Tap every tab | Jumps to the right section; the active tab is highlighted |
| 4 | Camera allowed | Start, allow camera, cover lens and flash | "พบนิ้ว ✓" (finger found) → settling → 20 s countdown → live waveform → result with quality score |
| 5 | Camera denied | Start, then deny permission | Red message explaining how to re-enable the camera, with retry and demo buttons |
| 6 | No camera / unsupported | Desktop without a camera, or an old browser | "ไม่พบกล้อง" (camera not found) or "ไม่รองรับ" (not supported), with the demo offered |
| 7 | Insecure context | Open over `http://` on a LAN IP | Message explaining HTTPS is required |
| 8 | Finger lifted mid-measurement | Remove finger for more than 1 s | "นิ้วหลุด…" (finger slipped); measurement restarts |
| 9 | Poor signal | Move the finger, or press very hard | "วัดไม่สำเร็จ" (measurement failed) with reasons; **no number shown** |
| 10 | Tab switch | Switch apps during measurement | Measurement stops and the camera is released |
| 11 | Demo mode | Tap "ลองโหมดสาธิต" (try demo mode) | Purple striped panel; result says "ข้อมูลจำลอง" (simulated data) with the target and computed BPM |
| 12 | Emergency symptoms | Tick chest pain, or breathing difficulty, or sudden weakness (each alone) | Red level-3 banner with a 1669 call button; mascot looks worried |
| 13 | Combinations | Fever + abdominal pain; fever + age 60+; cough + >2 weeks | Level 2 (orange) |
| 14 | Mild | Runny nose only | Level 0 (green) with self-care advice |
| 15 | Free text | Type "ไม่มีไข้ แต่เจ็บคอ" (no fever, but sore throat) → ค้นหา (search) | Only sore throat ticked |
| 16 | Empty submit | Press ดูคำแนะนำ (get advice) with nothing selected | Inline error; focus moves to the first checkbox |
| 17 | Thai TTS available | Android Chrome / iOS Safari: tap ฟังน้องสุขใจ (hear น้องสุขใจ), then อ่านผลออกเสียง (read result aloud) | Thai speech; mouth animates; หยุดอ่าน (stop reading) works |
| 18 | Thai TTS missing | Desktop without a Thai voice | Message explaining how to install one; status shown in วิธีใช้ (how to use) |
| 18a | Chat | Open "คุยกับน้องสุขใจ" (chat with น้องสุขใจ); type "สวัสดี" (hello), "ปวดหัว" (headache), then "แน่นหน้าอก" (chest tightness) | Mascot walks in and waves; headache → offer to check; chest tightness → red reply with 1669 and a worried face |
| 18b | Chat actions | Tap "ประเมินอาการนี้" (assess this symptom) / "ไปวัดชีพจร" (go measure pulse) | The chat closes on mobile and jumps to the section; the symptom form is filled and submitted |
| 18c | Mascot | Tap the mascot on the home screen several times | It cycles through excited (acorn), surprised, curious and waving, and the speech bubble text changes |
| 19 | Keyboard | Tab through the whole page; use Space on checkboxes and Enter on buttons | Visible focus everywhere; all actions reachable |
| 20 | Screen reader | TalkBack or VoiceOver over the results | Status updates announced; result heading read |
| 21 | Reduced motion | Turn on OS "reduce motion" | Mascot and scrolling animations stop |
| 22 | Offline | Load once, enable airplane mode, reload from cache or file | Everything except hosting works; no external requests |

## PPG validation protocol and table

**Protocol (suggested):** use a CE- or FDA-cleared fingertip pulse oximeter on one hand and the app on the other. Have the participant seated and at rest for 5 minutes. Record the oximeter's mean reading during the app's 20 s window. Include at least 20 participants and a range of skin tones, ages and phone models. Record flash use and the lighting conditions. Get consent, and do not record identifying data.

Use `ppg_validation_template.csv`, or the in-app "โหมดทดสอบความแม่นยำ" (accuracy test mode) table, which exports CSV. **Results are deliberately left blank until real measurements are collected.**

| # | Date-time | Participant ID | Device / browser | Flash | App BPM | Reference BPM | Error (App − Ref) | \|Error\| | Quality | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | | | | | | | | | | |
| 2 | | | | | | | | | | |
| … | | | | | | | | | | |
| **N** | | | | | | | | | | |

**Formulas** (spreadsheet rows 2–31, columns F = app, G = reference):

| Metric | Definition | Spreadsheet formula |
|---|---|---|
| Error (bias per row) | eᵢ = Appᵢ − Refᵢ | `=IF(AND(ISNUMBER(F2),ISNUMBER(G2)),F2-G2,"")` |
| Absolute error | \|eᵢ\| | `=IF(H2="","",ABS(H2))` |
| N | number of complete pairs | `=COUNT(I2:I31)` |
| **MAE** | (1/N) Σ \|eᵢ\| | `=IFERROR(AVERAGE(I2:I31),"")` |
| Mean bias | (1/N) Σ eᵢ | `=IFERROR(AVERAGE(H2:H31),"")` |
| Max absolute error | max \|eᵢ\| | `=IF(COUNT(I2:I31)=0,"",MAX(I2:I31))` |
| Share within ±5 BPM | count(\|eᵢ\| ≤ 5) / N | `=IFERROR(COUNTIF(I2:I31,"<=5")/COUNT(I2:I31),"")` |

Also report the **refusal rate**: the share of attempts where the app showed no number. A low MAE achieved by refusing most attempts is not a usable product.

## Remaining limitations

1. **PPG accuracy on real people is unknown.** It has only been tested on synthetic signals. Real-world factors not yet measured include skin tone, cold fingers, pressure, camera auto-exposure and white balance (the app cannot lock them), and rolling-shutter effects.
2. **The flash works mainly on Android Chrome.** On iPhone, and on laptops with only a webcam, the signal will often be too weak, and the app will correctly refuse to show a number.
3. **Irregular rhythms** (for example atrial fibrillation) will usually be refused or mis-estimated. The app does not and must not detect arrhythmia.
4. **Symptom rules were written by the dev team** from general public-health guidance. **A Thai clinician must review** the thresholds, wording and red-flag list before any real user testing.
5. **Keyword matching is naive:** it uses substring search with simple negation. It will miss spelling variants, dialect and indirect phrasing. The always-visible 1669 button is the safety net.
6. **TTS quality depends on the device.** Desktop Windows and Chrome often have no Thai voice.
7. Validation rows are not saved permanently; this is by design.
8. **The chat is keyword-based, not conversational AI.** Phrasing outside its lists gets the fallback reply. Thai has no word spaces, so short keywords can hide inside longer words; the intent order is tuned for known cases ("สบายดี" contains "บาย") but more will exist.
9. The mascot is a vector re-drawing of the design sheet, not the original artwork. Fine details (fur tufts, the back and side views) are simplified.

## Recommended next steps

1. Run the validation protocol: at least 20 participants, three or more phone models, flash on and off. Publish the MAE, bias, Bland–Altman plot and refusal rate.
2. Have a physician or nurse review `SymptomCore` (rules and Thai wording). Record that review in the README.
3. Run usability tests with five or more older adults. Measure task completion for: measure pulse, check symptoms, find 1669.
4. Use `ImageCapture`/`applyConstraints` to lock exposure and white balance where supported. Show a live "press more lightly" hint based on red saturation.
5. Optionally add a service worker so the app is installable and fully offline.
6. Before any public pilot, review Thailand's PDPA obligations, and whether medical-device rules from the Thai FDA (อย.) apply.
