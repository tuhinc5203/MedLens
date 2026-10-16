# MedLens

**Point your camera at your pill bottles and find out which ones shouldn't be mixed.**

MedLens reads the labels from a photo, lets you confirm what it found, and checks every pair
of medicines for known interactions. Each warning quotes the sentence from the official FDA
drug label it came from, so nothing is a black box.

> MedLens began as a hackathon idea. It was built in a day and never presented. It is a
> proof of concept, **not medical advice**. Use prop labels, not real prescription labels.

[See screenshots](screenshots/README.md) · [MIT License](LICENSE)

## Why it exists

People on several medicines often don't know that two of them interact, and the warning is
buried in dense clinical text on a label they will never read. MedLens turns a quick photo into
a plain-language answer, and it shows its evidence so the user (or a pharmacist) can verify it.

## Run it yourself

You need Python 3.10 or newer (tested on 3.11). No API keys, accounts or database are required.

```bash
git clone https://github.com/tuhinc5203/MedLens.git && cd MedLens
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/streamlit run streamlit_app.py
```

On Windows, use `.venv\Scripts\python` and `.venv\Scripts\streamlit` instead.

Streamlit opens <http://localhost:8501> in your browser.

**No camera, or camera blocked on your machine?** Skip the photo step: choose a demo bottle or
use **Type my list** on the first screen. Everything after that (confirming, interaction
checks, the cabinet) works the same.

### Try it with your laptop camera

1. Click **Use webcam** and allow camera access when the browser asks.
2. Hold a medicine bottle to the camera with the label facing it, and take the photo.
3. Review the list MedLens read, fix or remove anything wrong, and confirm.
4. See which pairs interact, with the FDA label sentence behind each flag.

No bottles handy? Print or draw a prop label (a name and strength is enough, for example
"Ibuprofen 200 mg").

### Try it with your phone camera

Run the app on your laptop, then open the **Network URL** that Streamlit prints
(something like `http://192.168.1.20:8501`) in your phone's browser. Your phone and laptop must
be on the same Wi-Fi.

Tap **Take or choose photo**. On a phone this opens the native camera (rear lens, full
quality). Use this button rather than **Use webcam** on a phone: browsers only allow
live camera access on `localhost` or HTTPS, and the phone is connecting over plain HTTP.

### Optional: better label reading

Out of the box, MedLens reads photos with on-device OCR. For curved or glossy labels you can
switch to a vision model. MedLens picks whichever is available:

| Backend | Setup | Where the photo goes |
| --- | --- | --- |
| On-device OCR (default) | none | stays on your machine |
| Ollama | `ollama pull qwen2.5vl:7b` and leave Ollama running | stays on your machine |
| Gemini | `export GEMINI_API_KEY=...` (free key: <https://aistudio.google.com/apikey>) | sent to Google |

### Run the tests

```bash
.venv/bin/python -m pytest -q
```

The tests run offline against the committed cache in `data/cache/`.

## How it works

1. **Read** ([medlens/vision.py](medlens/vision.py)): a vision model (Gemini or Ollama) or
   on-device OCR (RapidOCR) transcribes each label. Unreadable fields come back empty, never
   guessed. With OCR, RxNav decides which lines are drug names, so text like "(NSAID)" or
   "every morning" never becomes a medicine.
2. **Confirm**: you fix or remove anything wrong before any advice is produced.
   Low-confidence readings are flagged.
3. **Resolve** ([medlens/rx.py](medlens/rx.py)): [RxNav](https://rxnav.nlm.nih.gov) turns label
   text ("Metoprolol Succ ER 25mg", "Mobic") into active ingredients and drug classes. A match
   must resemble a word actually printed on the label, otherwise it is "not recognised".
4. **Check** ([medlens/check.py](medlens/check.py)): for every pair, look for the other drug
   (by name, brand or class, e.g. "NSAIDs", "SSRIs") in each drug's
   [openFDA](https://open.fda.gov/apis/drug/label/) label interaction text. The matching
   sentence is kept as evidence and the severity is read from its wording ("contraindicated" /
   "avoid" is high, "monitor" / "may increase" is moderate). Negations such as "no dose
   adjustment needed" are ignored. Two products with the same active ingredient are flagged as
   possible double dosing.
5. **Explain** ([medlens/advice.py](medlens/advice.py)): fixed, plain wording. It never tells
   anyone to stop a medicine. A "space your doses apart" note appears only when the label itself
   says so (for example levothyroxine and calcium), and says spacing *won't* help when the label
   says the effect happens even 12 hours apart (clopidogrel and omeprazole).

### My Cabinet and reminders

Save medicines from a scan (or add them by hand) to a running **Cabinet**, give each one daily
dose times, and tick doses off in a **Today** checklist. **Check my whole cabinet** runs the
interaction check across everything saved, and a newly scanned bottle can be checked against the
cabinet too.

- **Private by design:** the cabinet lives in your browser's localStorage
  ([components/cabinet_store](components/cabinet_store)). It is never sent to a server, and
  another device or browser starts empty.
- **Calendar reminders:** *Add daily reminders to my calendar* downloads an `.ics` file with a
  daily-repeating event and an alert at each dose time. It works in Apple, Google and Outlook
  calendars, so your phone alerts you even when the app is closed.
- **Optional in-app notifications:** a *Notify me at dose times* switch (off by default) shows a
  browser notification at each dose time, skipping doses already ticked off. It only works while
  MedLens is open in a tab, and not in iOS Safari outside an installed web app. True background
  push would need accounts and a server, which this project deliberately avoids.

## Project layout

```
streamlit_app.py          The Streamlit UI (scan, confirm, results, cabinet)
medlens/
  vision.py               Photo to label text (Gemini, Ollama or on-device OCR)
  rx.py                   Label text to active ingredients and drug classes (RxNav)
  check.py                Pairwise interaction search over FDA label text
  advice.py               Plain-language wording for each result
  cabinet.py              Cabinet data, dose schedule and .ics reminder export
  store.py                Drug data fetching and caching (openFDA, RxNav)
  store_browser.py        Bridge to the browser-side cabinet storage
components/cabinet_store/ Browser localStorage and notification component
data/cache/               Pre-fetched drug data so common drugs work offline
scripts/build_cache.py    Rebuilds the cache for about 130 common drugs
tests/                    Offline test suite
```

Common drugs answer instantly from `data/cache/`. Any other drug is fetched live from RxNav
and openFDA on first use. Set `MEDLENS_OFFLINE=1` to forbid network access and use only the cache.

## Limits (please read)

- **Severity is a reading of label wording, not a clinical model.** Labels are written for
  clinicians, and a sentence match can miss an interaction or over-read a generic sentence.
  Anything unusual should go to a pharmacist, which is what the app tells users.
- Only interactions the FDA label text mentions are found. Herbal products, foods, doses and a
  patient's conditions are not considered.
- Drugs with no label interaction text, drugs the name lookup can't resolve, and unreadable
  bottles are listed under **Not checked**; they are never silently dropped.
- Photos are sent to Google when the Gemini backend is used. Use prop labels.

## Background

MedLens was a hackathon idea, written from an empty directory in a single day and never
presented. The only data inputs are public: RxNav (NLM) and openFDA drug labels.
