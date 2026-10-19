# Logbook

Turns photographed paper donation forms into spreadsheet rows. Built for a food bank that logs in-kind donations on paper intake forms and re-types them into Excel each month.

Staff photograph a stack of forms on a phone. Claude reads each photo and fills in the fields. A person reviews every form, with mismatched weights and likely duplicates flagged, then saves the batch, and the rows are inserted into the right monthly tab of the year's spreadsheet.

In a timed run, a 50-form batch took about 15 minutes, against the food bank's own estimate of about 2 hours by hand.

## How it works

```
Phone (Expo / React Native)                Laptop on the same Wi-Fi (Flask)
---------------------------                --------------------------------
Camera tab: photograph forms
  each photo -> form record (unscanned)
  stored locally in AsyncStorage
                    -- POST /ocr (base64 JPEG) -->  Claude Haiku 4.5 vision
                    <-- JSON fields --------------  JSON-only extraction prompt
Review tab: form -> needs_review or ocr_failed
  staff correct and approve each form
  flags: category weights don't add up to total (> 0.5 lb off),
         consecutive forms with identical fields (possible duplicate)
Save tab: approved forms
                    -- POST /append ------------->  openpyxl: year file, month tab,
                                                    date-ordered insert, SUM totals
```

- **Status flow:** `unscanned` -> `needs_review` | `ocr_failed` -> `approved`. Failed forms reset to `unscanned` and retry when staff refresh.
- **Parallel OCR:** photos are sent as they are taken. Each form update goes through a promise queue, so concurrent OCR results don't overwrite each other.
- **The model leaves a field blank when it can't read it.** Nothing is saved until a person approves it.
- **Output matches the food bank's existing spreadsheet:** same columns, perishable and non-perishable totals, and a monthly SUM row, so their reporting didn't change.

## Run it

**Server** (Python 3.10+):

```bash
cd server
pip install -r requirements.txt
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env
python server.py          # listens on 0.0.0.0:5000
```

Spreadsheets are written to `server/data/` (git-ignored).

**App:**

```bash
npm install
npx expo start
```

Open it in Expo Go on a phone on the same network. On the Save tab, set the server URL to your laptop's LAN address, for example `http://192.168.1.42:5000`.

## Known limitations

This is a prototype built in about ten days for one site.

- The server has no authentication and runs Flask in debug mode. Run it only on a trusted local network.
- Claude's JSON reply is parsed but not validated against a schema. Structured output or tool use would be the fix.
- `/append` isn't idempotent: retrying a save writes the rows again.
- Forms live on a single phone's local storage. There is no multi-device sync.
