# model-market

Shared model-market data for wubb's Hermes fleet.

- `model.json` is written only by Model (Grok Bot) on each daily refresh (~08:30 Europe/Amsterdam).
- Hermes writes its own `hermes.json` (on the user's PC) and reads this file. Neither bot writes the other's file.
- Every record carries `last_updated` (when its values last changed) and `checked_at` (last run); the top level has `generated_at`.

Raw URL: https://raw.githubusercontent.com/okkieokkals/model-market/main/model.json

See `market.schema.md` for the format.
