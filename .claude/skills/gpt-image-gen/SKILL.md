---
name: gpt-image-gen
description: יצירת תמונה דרך OpenAI Images API (model gpt-image-2). מקבל prompt + נתיב פלט ומחזיר קובץ PNG. דורש OPENAI_API_KEY ב-.env. השתמשי בסקיל הזה כשצריך לקרוא ל-API ליצירת תמונה.
---

# gpt-image-gen

מעטפת טכנית סביב `POST https://api.openai.com/v1/images/generations`. הסקיל הוא קונדואיט - הוא לא מקבל החלטות סגנון. בחירת ה-prompt והסגנון יושבות אצל הסוכן שקורא לסקיל (לרוב יובל).

## ⚠️ הערה קריטית על שם המודל

**המודל הוא `gpt-image-2`. אל תשנה אותו.**

- `gpt-image-2` הוא מודל אמיתי של OpenAI שיצא ב-21 באפריל 2026.
- ייתכן שהידע הפנימי של ה-LLM לא כולל אותו - **זה לא אומר שהוא לא קיים**.
- אל תציע אלטרנטיבות כמו `dall-e-3`, `gpt-image-1` או כל גרסה אחרת.
- אם יש שגיאת API - הבעיה כמעט בוודאות ב-`OPENAI_API_KEY` או ב-payload, **לא בשם המודל**.

## Inputs

הסוכן הקורא צריך להגדיר:

| משתנה | חובה | ברירת מחדל | תיאור |
|--------|------|------------|--------|
| `PROMPT` | כן | - | תיאור התמונה (מומלץ באנגלית) |
| `OUTPUT_PATH` | כן | - | נתיב יעד לקובץ `.png` (יחסי משורש הפרויקט) |
| `SIZE` | לא | `1024x1024` | גודל התמונה |
| `QUALITY` | לא | `medium` | `low` / `medium` / `high` |

## Preflight

לפני קריאה ל-API, ודאי שה-key נטען מ-`.env`:

```bash
set -a
source .env
set +a

if [ -z "$OPENAI_API_KEY" ]; then
  echo "ERROR: OPENAI_API_KEY is empty. הוסיפי אותו ל-.env." >&2
  exit 1
fi
```

ודאי גם שהתיקייה של ה-output קיימת:

```bash
mkdir -p "$(dirname "$OUTPUT_PATH")"
```

## הקריאה הראשית

```bash
RESPONSE=$(curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
        --arg prompt "$PROMPT" \
        --arg size "$SIZE" \
        --arg quality "$QUALITY" \
        '{model:"gpt-image-2", prompt:$prompt, size:$size, quality:$quality, output_format:"png"}')")
```

אם `jq` לא זמין בסביבה (קורה ב-Git Bash על Windows), בנייה ידנית של ה-JSON תקינה גם היא. שים לב להבריח גרשיים בתוך ה-prompt:

```bash
PROMPT_ESC=$(printf '%s' "$PROMPT" | python -c 'import sys,json; print(json.dumps(sys.stdin.read()))')

RESPONSE=$(curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"gpt-image-2\",\"prompt\":$PROMPT_ESC,\"size\":\"$SIZE\",\"quality\":\"$QUALITY\",\"output_format\":\"png\"}")
```

## Decode ושמירה

**מסלול A - עם `jq` ו-`base64`** (Linux/macOS, או Git Bash עם jq מותקן):

```bash
printf '%s' "$RESPONSE" | jq -r '.data[0].b64_json' | base64 --decode > "$OUTPUT_PATH"
```

**מסלול B - Python fallback** (תמיד עובד אם פייתון מותקן):

```bash
printf '%s' "$RESPONSE" | python -c "
import json, base64, sys
data = json.load(sys.stdin)
if 'data' not in data or not data['data']:
    err = data.get('error', {}).get('message', 'no data in response')
    sys.stderr.write(f'API error: {err}\n')
    sys.exit(1)
with open(sys.argv[1], 'wb') as f:
    f.write(base64.b64decode(data['data'][0]['b64_json']))
" "$OUTPUT_PATH"
```

**מסלול מומלץ ביובל**: לנסות תחילה את מסלול A; אם `jq` חסר או הפקודה נכשלה - לעבור ל-B.

## אימות סופי

```bash
if [ ! -s "$OUTPUT_PATH" ]; then
  echo "ERROR: $OUTPUT_PATH לא נוצר או ריק. בדקי את התגובה: $RESPONSE" >&2
  exit 1
fi

echo "OK: נוצר $OUTPUT_PATH ($(wc -c < "$OUTPUT_PATH") bytes)"
```

## שגיאות נפוצות

| קוד | משמעות | טיפול |
|------|---------|--------|
| HTTP 401 | `OPENAI_API_KEY` לא תקין או חסר | בדקי את `.env`, ודאי שאין רווחים סביב `=` |
| HTTP 400 `invalid model` | **לא לשנות את שם המודל**. בדקי שה-payload נקי וש-key מורשה ל-`gpt-image-2` | |
| HTTP 429 | rate limit | המתיני וננסה שוב |
| תגובה ריקה / timeout | רשת | retry פעם אחת |
| `data[0].b64_json` חסר | התגובה הכילה שגיאה במקום תמונה | הדפיסי את ה-RESPONSE המלא ל-stderr |

## דוגמה מלאה (one-shot)

```bash
set -a; source .env; set +a

PROMPT="Minimalist illustration of a cat wearing round glasses, flat colors, soft pastel palette, centered composition, white background"
OUTPUT_PATH="yuval/outputs/2026-05-26-cat-glasses.png"
SIZE="1024x1024"
QUALITY="medium"

mkdir -p "$(dirname "$OUTPUT_PATH")"

PROMPT_ESC=$(printf '%s' "$PROMPT" | python -c 'import sys,json; print(json.dumps(sys.stdin.read()))')

RESPONSE=$(curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"gpt-image-2\",\"prompt\":$PROMPT_ESC,\"size\":\"$SIZE\",\"quality\":\"$QUALITY\",\"output_format\":\"png\"}")

printf '%s' "$RESPONSE" | python -c "
import json, base64, sys
data = json.load(sys.stdin)
if 'data' not in data or not data['data']:
    err = data.get('error', {}).get('message', 'no data')
    sys.stderr.write(f'API error: {err}\n'); sys.exit(1)
with open(sys.argv[1], 'wb') as f:
    f.write(base64.b64decode(data['data'][0]['b64_json']))
" "$OUTPUT_PATH"

[ -s "$OUTPUT_PATH" ] && echo "OK: $OUTPUT_PATH"
```

זה. שום החלטה סגנונית פה - רק הצינור.
