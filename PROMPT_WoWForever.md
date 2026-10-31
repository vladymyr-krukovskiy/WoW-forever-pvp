# Промпт: аддон BGForeverStatisticShow для WoW Forever

Скопіюй усе нижче лінії в нову сесію Claude Code (у порожній папці майбутнього аддона).

---

Ти — досвідчений розробник аддонів для World of Warcraft. Створи аддон **BGForeverStatisticShow** для клієнта **WoW Forever** (внутрішня назва — Camelot). Це порт мого аддона BGWOWClassicStatisticShow (TBC Classic) на новий клієнт, з виправленням відомих недоліків оригіналу.

## 0. ОБОВ'ЯЗКОВО: налаштування MCP перед будь-яким кодом

WoW Forever використовує сучасний (Mainline, ~12.1.x) API, а не класичний. Багато старих функцій видалено або змінено, тому **не пиши код з пам'яті** — кожну функцію, подію, шаблон і mixin перевіряй через MCP-сервер **hated-wow-mcp** (MIT, https://github.com/RdyGaming/hated-wow-mcp, потрібен Node 20+).

1. Перевір, чи сервер уже підключений (`claude mcp list`). Якщо ні — додай його з flavor `forever` за замовчуванням:
   ```bash
   claude mcp add wow -e WOW_DEFAULT_FLAVOR=forever -- npx -y hated-wow-mcp
   ```
2. Синхронізуй вихідний код Blizzard UI для Forever (потрібно для `wow_ui_*` інструментів):
   ```bash
   npx -y hated-wow-mcp sync ui-source -- forever
   ```
3. Перезапусти сесію, якщо інструменти `wow_*` ще не з'явились, і виклич `wow_data_status`, щоб переконатися, що дані для `forever` доступні.
4. **Якщо MCP не вдалося підключити — зупинись і повідом мене.** Не продовжуй без нього.

Правила використання MCP протягом усієї роботи:
- Перед використанням будь-якої API-функції — `wow_api_search` (flavor `forever`). Перед підпискою на подію — `wow_api_event_search` (перевір payload).
- Для шаблонів frame-ів і mixin-ів (рамки, чекбокси, слайдери, dropdown) — `wow_ui_template_search` / `wow_ui_mixin_search`, при потребі `wow_ui_read_file`.
- Для іконок класів/фракцій — `wow_atlas_search` / `wow_icon_search` (віддавай перевагу atlas замість старих шляхів текстур).
- Каркас створи через `wow_addon_scaffold`, шлях установки дізнайся через `wow_install_info`.
- Після кожної суттєвої зміни: `wow_lua_lint` для всіх `.lua` і `wow_toc_validate` для `.toc`.
- Якщо функції немає в API Forever — знайди заміну через `wow_api_search` / `wow_api_diff`, не вигадуй.

## 1. Функціональність (як в оригіналі)

Невелике переміщуване вікно, що показує статистику гравця на поточному полі бою:
- **Шапка:** іконка класу, ім'я гравця кольором класу, іконка фракції (Horde/Alliance) справа.
- **Таблиця з одного рядка, 4 колонки:** Killing Blows, Honorable Kills, Deaths, Honor Gained.
- **Оновлення:** на полі бою — раз на 3 секунди запит даних скорборду + реакція на подію оновлення скорборду. Поза БГ таймер зупиняється.
- **Перетягування** лівою кнопкою, вікно не виходить за межі екрана, позиція зберігається між сесіями.
- **Налаштування** (панель у стандартному Settings UI гри):
  - вибір вигляду рамки (кілька варіантів шаблонів/backdrop, перелік перевір через MCP — старі `GlowBoxTemplate`, `InsetFrameTemplate`, `InsetFrameTemplate3`, `TranslucentFrameTemplate` можуть бути відсутні);
  - чекбокс «Показувати поза полем бою»;
  - слайдер масштабу 50–200 %, крок 5 %.

## 2. Що виправити порівняно з оригіналом

1. **Скидати статистику в нулі** при вході на нове поле бою та при виході з нього (оригінал показував застарілі цифри).
2. **Не створювати новий frame при зміні вигляду рамки.** Один постійний frame; змінюється лише фон/рамка (дочірній frame-підкладка або backdrop).
3. **Позиція не повинна зсуватися при зміні масштабу** (зберігай/відновлюй позицію з урахуванням `GetScale`/`GetEffectiveScale`).
4. **Надійний пошук себе в скорборді:** порівнюй за GUID, якщо API скорборду його повертає; інакше — за іменем з відрізаним суфіксом реалму. Враховуй, що фільтр фракції в скорборді може приховати гравця.
5. Ключі статистики задай упорядкованим списком і будуй таблицю циклом, без ланцюжків `if j == 1 … elseif`.
6. Додай slash-команди `/bgfs` (відкрити налаштування), `/bgfs reset` (скинути позицію), `/bgfs lock` (заблокувати/розблокувати перетягування) і чекбокс «Заблокувати вікно» в налаштуваннях.
7. Тексти інтерфейсу винеси в таблицю локалізації (`enUS` за замовчуванням + `ukUA`/`ruRU`, якщо клієнт підтримує відповідну локаль — перевір `GetLocale`).

## 3. Технічні вимоги для WoW Forever

- **TOC:** `## Interface: 16001` (значення з бета-версії — перед завершенням звір через `wow_toc_validate` та `wow_install_info`; якщо є актуальніше — використай його). Додай `## Title`, `## Notes`, `## Author`, `## Version`, `## SavedVariables: BGForeverStatisticShowDB`, `## IconTexture` або `## IconAtlas`. Розглянь окремий файл `BGForeverStatisticShow_Camelot.toc`, якщо MCP підтверджує цей суфікс.
- **Lua 5.1:** без `goto`, `//`, бітових операторів, `require`/`io`/`os`. Кожен файл починай з `local addonName, ns = ...`, спільний стан — у `ns`, мінімум глобалів.
- **SavedVariables** читай в `ADDON_LOADED` (для свого аддона), заповнюй дефолти, валідуй значення (наприклад, неіснуючий шаблон → дефолтний).
- **Скорборд БГ:** у Mainline API класичні `GetBattlefieldScore` / `GetNumBattlefieldScores` / `RequestBattlefieldScoreData` можуть бути замінені на `C_PvP.*` (наприклад `C_PvP.GetScoreInfo`, `C_PvP.GetScoreInfoByPlayerGuid`) — **визнач актуальні функції та подію оновлення через MCP** і використовуй саме їх.
- **Визначення поля бою:** `IsInInstance()` / `C_PvP.IsBattleground()` або аналог — перевір через MCP.
- **Settings UI:** використовуй `Settings.RegisterVerticalLayoutCategory` / `Settings.RegisterAddOnSetting` / `Settings.CreateCheckbox` / `Settings.CreateSlider` / `Settings.CreateDropdown` (або canvas-категорію, якщо так простіше). **Не використовуй** `InterfaceOptions_AddCategory`, `InterfaceOptionsCheckButtonTemplate`, `OptionsSliderTemplate`, `UIDropDownMenu*` — перевір через MCP, що вибрані API існують у Forever.
- **Іконки:** клас — atlas на кшталт `classicon-<class>` або `C_Texture`/`CLASS_ICON_TCOORDS` (що реально існує — через MCP); фракція — відповідний atlas.
- **Secret values:** у Forever діють обмеження Midnight. Деякі значення (імена юнітів, здоров'я тощо) можуть бути «секретними». Якщо значення, отримані зі скорборду чи `UnitName`, можуть бути secret — перевіряй через `issecretvalue` і не порівнюй/не конкатенуй їх напряму.
- **Не використовуй** `COMBAT_LOG_EVENT_UNFILTERED` / `CombatLogGetCurrentEventInfo` — у Forever вони недоступні.
- Не чіпай захищені frame-и Blizzard; зміни, що можуть зачепити захищений код, роби поза боєм (`InCombatLockdown()` → відкласти до `PLAYER_REGEN_ENABLED`).
- `WOW_PROJECT_ID` не використовуй як єдину ознаку клієнта — перевіряй наявність потрібного API.

## 4. Структура проєкту

```
BGForeverStatisticShow/
  BGForeverStatisticShow.toc
  Locales.lua      -- таблиця рядків
  Core.lua         -- БД, події, скорборд, таймер
  UI.lua           -- головне вікно, таблиця, іконки, перетягування
  Options.lua      -- Settings UI та slash-команди
  README.md        -- опис, встановлення, налаштування, відомі обмеження
```

## 5. Порядок роботи та критерії готовності

1. Виконай розділ 0 (MCP) і покажи мені результат `wow_data_status`.
2. Через MCP склади й покажи мені **таблицю використаних API** (функція/подія/шаблон → підтверджено для `forever` так/ні → чим замінено). Тільки після цього пиши код.
3. Напиши аддон за структурою вище.
4. Прожени `wow_lua_lint` по всіх Lua-файлах і `wow_toc_validate` по TOC — 0 помилок.
5. Виведи, куди скопіювати папку (через `wow_install_info`; для бети зазвичай `World of Warcraft\_classic_beta_\Interface\AddOns\`, для релізу — `_forever_`), і чекліст ручної перевірки в грі:
   - вікно з'являється, ім'я в кольорі класу, іконки коректні;
   - на БГ цифри оновлюються, після виходу скидаються;
   - перетягування, збереження позиції після `/reload`, зміна масштабу без зсуву;
   - усі налаштування працюють і зберігаються;
   - `/bgfs`, `/bgfs reset`, `/bgfs lock`;
   - немає Lua-помилок (`/console scriptErrors 1`).
