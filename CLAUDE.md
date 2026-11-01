# WoW Forever — заметки для разработки аддонов

Собрано 2026-11-01, пока WoW Forever в бете. **При релизе перепроверить** номер Interface, теги библиотек и список недоступных API.

Подробный промпт для аддона BGForeverStatisticShow — в [PROMPT_WoWForever.md](PROMPT_WoWForever.md).

## Клиент
- Тип игры «camelot»: `WOW_PROJECT_ID == WOW_PROJECT_CAMELOT` (18), не `WOW_PROJECT_MAINLINE`.
- API — Mainline (ретейл), Lua 5.1.
- `.toc`: `## Interface: 16001` (значение беты, релизное ещё неизвестно).
- Аддоны беты ставятся в `_classic_beta_\Interface\AddOns`.

## MCP
- **hated-wow-mcp** (основной, есть flavor `forever`) — https://github.com/RdyGaming/hated-wow-mcp, нужен Node 20+:
  ```bash
  claude mcp add wow -e WOW_DEFAULT_FLAVOR=forever -- npx -y hated-wow-mcp
  npx -y hated-wow-mcp sync ui-source -- forever
  ```
  Проверка: `wow_data_status`. Не писать API по памяти — проверять через `wow_api_search` / `wow_api_event_search`, после изменений `wow_lua_lint` и `wow_toc_validate`.
- **wow-api-mcp (wutname1)** — сигнатуры 8000+ функций, deprecated-список; требует VS Code-расширение `ketho.wow-api`.
- **sylvanas-docs-mcp** — документация ретейл API со снимками по патчам и диффами.

## Библиотеки (проверены на Forever)
- **Ace3 — брать из trunk**: тег r1403 не знает 16001 и не содержит фиксов под Forever.
- ChatThrottleLib, LibSerialize, LibDeflate, LibDataBroker, LibStub.

## Инструменты
- **BigWigs packager** v2.6.0+ — поддерживает Forever, для сборки релизов.
- **forever-addon-kit** — https://github.com/Thunderz96/forever-addon-kit: дамп API из игры, скрипты портирования, `api_scan.py` / `api_scan_all.py` (находят вызовы, которых нет в клиенте Forever). Прогонять перед релизом.
- VS Code + `ketho.wow-api` (аннотации LuaLS).
- Примеры аддонов: https://github.com/topics/wow-forever

## Подводные камни беты
- `ReloadUI()` защищена — кнопки «Reload UI» в аддонах не работают.
- Триальные аккаунты не могут отправлять addon-сообщения.
- Многих ретейл-функций нет — каждый вызов защищать проверкой: `if C_Something and C_Something.Func then ... end`.
