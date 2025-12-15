# EPL predictor helper

This repository contains a small utility that reads a table with users' match predictions,
compares it with the actual match results and produces an Excel standings sheet that looks
like the sample on the screenshot from the request. Every participant receives points per
match according to the rules:

1. 4 points — guessed the full score (both teams goals).
2. 2 points — guessed the winner and the goal difference but not the exact score.
3. 1 point — guessed only the winner (or a draw).

The standings sheet lists the overall totals (place, user name, number of predictions,
exact scores, total points, average per round) and then adds two columns per round:
`Round N exact` and `Round N points`. You can edit the column names in the script if you
want to localize the table — only ASCII is used by default.

## Project layout

```
├── data
│   ├── predictions_sample.csv
│   ├── results_sample.csv
│   ├── raw_results_template.txt
│   └── raw_predictions_template.txt
├── output
│   └── apl_standings.xlsx        # created after you run the script once
├── requirements.txt
└── scripts
    ├── generate_scoreboard.py
    ├── import_text_results.py
    ├── import_text_predictions.py
    └── update_from_text.py
```

## Input files

Both the prediction file and the results file can be in CSV/TSV/XLS/XLSX format. The
script detects the format by extension and expects the following column names:

### Predictions

| column | description |
| --- | --- |
| `match_id` | Identifier that matches the results table. |
| `round` | Tour/round identifier (integer). |
| `user_id` | Optional stable identifier for a participant (used together with `user`). |
| `user` | Participant name. |
| `home_team`, `away_team` | Optional metadata (used only if the results file misses them). |
| `predicted_home_goals`, `predicted_away_goals` | Integer score prediction. |

### Results

| column | description |
| --- | --- |
| `match_id` | Identifier that matches the prediction table. |
| `round` | Tour id (integer). |
| `home_team`, `away_team` | Team names. |
| `home_goals`, `away_goals` | Real match score. |

Add as many rows/users as you need — the scoring logic is data-driven.

## Install dependencies

```bash
cd epl-predictor
python3 -m pip install -r requirements.txt
```

(pandas and openpyxl are already installed in many Python environments; skip the command
if you already have them.)

## Run the script

```bash
python3 scripts/generate_scoreboard.py \
  data/predictions_sample.csv \
  data/results_sample.csv \
  output/apl_standings.xlsx
```

Or use the helper that also imports the template text files:

```bash
./run_project.sh <round-number>
```

The script reads `data/raw_results_template.txt` and `data/raw_predictions_template.txt`,
imports their contents into `data/results_sample.csv` / `data/predictions_sample.csv`
(replacing predictions for the listed fixtures) and finally rebuilds
`output/apl_standings.xlsx`.

Arguments:

- `predictions`: path to the prediction file.
- `results`: path to the file with real scores.
- `output`: Excel workbook to create/update. If the workbook already exists, only the
  specified sheet is replaced.
- `--sheet`: optional sheet name (defaults to `Standings`).

The generated Excel file can be formatted manually in LibreOffice/Excel to match your
color scheme (like on the provided screenshot). Re-run the script to refresh the data
whenever new rounds finish — the sheet is recalculated from scratch every time.

### Import match results from a text file

When the real match results arrive as a simple text snippet (for example pasted from a
messenger) you can convert it to the CSV format used by the pipeline:

```bash
python3 scripts/import_text_results.py \
  raw_round.txt \
  data/results_sample.csv \
  --round 3
```

Each line should look like either `Ліверпуль 1 : 2 Брайтон` or `Ліверпуль - Брайтон 1:2`
(склад учасників і рахунок у кінці/у середині — обидва варіанти підтримуються). Any
other lines (emojis, participant names, separators) are ignored. New lines are appended
to the existing CSV, and the script automatically assigns sequential match ids (`M5`,
`M6`, ...). Команди можна писати українською, російською або англійською — скрипт
розпізнає найпоширеніші варіанти назв.

### Import user predictions from a text file

If you receive participants' predictions in chat format (a few meta lines followed by
scores), convert them into the structured `predictions_sample.csv` file:

```bash
python3 scripts/import_text_predictions.py \
  raw_predictions.txt \
  data/results_sample.csv \
  data/predictions_sample.csv \
  --clear-users
```

For every block of text, add metadata lines (user id, user name, etc.) before the match
list. The importer uses the results file to find the correct `match_id`, so team names
must match existing fixtures; Ukrainian, Russian and English spellings for EPL clubs are
normalised automatically. Add `--round` if all predictions should be tied to a
specific tour number, otherwise the script copies the round from the results file.
With `--clear-users`, only the touching fixtures for those users are replaced — their
older rounds remain in the CSV. If a user id is missing in the text, the importer assigns
an auto-generated identifier like `U0005` and reuses it for the same name next time.

### Выполнить оба шага одной командой

Чтобы не запускать скрипты по отдельности, используйте обёртку, которая сначала импортирует
текстовые результаты, а затем создаёт Excel:

```bash
python3 scripts/update_from_text.py \
  data/raw_results_template.txt \
  --round 3 \
  --predictions-text data/raw_predictions_template.txt \
  --predictions data/predictions_sample.csv \
  --results data/results_sample.csv \
  --output output/apl_standings.xlsx
```

Опция `--match-prefix` задаёт префикс для `match_id` (по умолчанию `M`), а `--sheet`
меняет имя листа в итоговом файле. `--predictions-text` добавляет обработку текстового
файла с прогнозами (используется та же структура, что и для `import_text_predictions.py`);
добавьте `--clear-predictions`, если нужно заменить прогнозы именно для указанных матчей —
история предыдущих туров при этом сохранится. После выполнения текстовые файлы можно
очистить или заменить новыми данными перед следующим запуском.

## Customizing the layout

- Add a `user_id` column to the prediction file if you want the standings to display and
  compare users by both name and id (useful when two participants share a name).
- Adjust scoring rules in `_row_points` inside `scripts/generate_scoreboard.py` if your
  league uses different scoring.
- Change the column names inside `_build_standings` if you prefer Ukrainian labels (e.g.
  "Місце", "Ім'я", ...).
- Extend `_write_excel` to add formatting or multiple sheets if you need richer reports.

The sample CSV files inside `data/` can be used to test the pipeline end-to-end.
