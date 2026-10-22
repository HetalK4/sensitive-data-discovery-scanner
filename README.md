# Sensitive Data Discovery Scanner

A scanner that audits a 100,000-record database for exposed personal data and credentials,
using regular expressions and custom YARA rules, and writes every finding to an audit log.

> **Independent side project.** The Big Data Security course set a group project. This is
> a separate version that I built on my own alongside it. It is **not the group's
> submitted work**, which is a different notebook and is not in this repository.

## What it does

1. **Builds a synthetic database.** 100,000 fake customer profiles (names, jobs and
   companies from [Faker](https://faker.readthedocs.io/)) are stored in a local SQLite
   table. Every 20th row (5,000 rows) gets one deliberately injected leak, chosen at
   random from: an email address, a phone number, a plain-text password, or an API key.
   **All data and credentials are fake.**
2. **Defines the detection rules.**
   - Regex for email addresses and phone numbers.
   - A YARA rule (`Sensitive_Credentials_Hunter`) for the phrase "password is ...",
     AWS-style keys (`AKIA...`) and Google-style keys (`AIzaSy...`).
3. **Scans every row**, counts each type of exposure, and logs every hit to
   `scan_activity.log` at WARNING (PII) or CRITICAL (credentials) level.
4. **Prints an exposure report** and runs a six-check test matrix.

## Results (from the saved run in the notebook)

| Measure | Count |
|---|---|
| Records scanned | 100,000 |
| Emails flagged | 1,226 |
| Phone numbers flagged | 1,244 |
| Secrets flagged (rows) | 1,688 |

Both `random` and Faker are seeded with 42, so a re-run should reproduce these counts
(a different Faker version can change the generated data). Emails and phones are counted
as matches. Secrets are counted as rows.

## What these results do and don't show

Counting detections is not the same as measuring accuracy, because the notebook does not
keep a record of which rows it injected. Working it out from the code:

- **Emails and phones:** about 1,250 of each were injected, and the counts above are
  close to that.
- **Credentials: roughly a third of the injected ones are missed.** About 2,500
  credentials were injected (about 1,250 passwords and 1,250 API keys), but only 1,688
  were flagged. The cause is a mismatch between the test data and the rules. The data uses
  three mock API-key formats, and only the AWS-style one is matched. The Google-style mock
  key is 37 characters, while the rule expects 39, and the third format has no rule at
  all. So roughly two thirds of injected API keys go undetected. I found this while
  writing this README, and it is not fixed yet.
- **The tests are sanity checks, not an accuracy measure.** The six checks cover two
  positive regex cases, one negative control (which only covers emails and YARA), two YARA
  cases and one log-file check.
- **No false-positive rate was measured.** The phone regex is broad and could match other
  digit strings in real data.
- **The data is synthetic.** Injected leaks have simple, regular formats. Real data is
  much messier.

## Next steps

- Record which rows were injected, so precision and recall can be calculated properly.
- Fix the Google-style mock key and add a rule for generic secrets.
- Tighten the phone regex and measure false positives.

## Running it

```bash
pip install -r requirements.txt
jupyter notebook sensitive_data_discovery_scanner.ipynb
```

Run the notebook top to bottom. It creates `company_data.db` and `scan_activity.log` in
the working directory. Both are git-ignored. `yara-python` may need build tools on some
systems.

## Tech stack

`Python` · `SQLite` · `Faker` · `YARA (yara-python)` · `regex` · `logging`

## License

Released under the MIT License. See `LICENSE`.
