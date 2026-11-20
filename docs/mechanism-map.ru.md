# Индекс кода механик

Этот файл отвечает только на вопрос **«из какого файла и какой функции входить в эту механику»**. Это таблица маршрутов, а не руководство:

- числовые значения геймплея и описание механик → `docs/gameplay.md` и тематические документы по механикам;
- ловушки, на которые можно наступить при правках → `specs/capabilities/`;
- почему когда-то было принято именно такое решение → `docs/adr/` и `specs/changes/<slug>/spec.md`.

Здесь указаны только файлы и входные функции, **без номеров строк и без пересказа правил** (номера строк устаревают, у правил есть единственный владелец). Всё поведение механик определяют код и тесты. Если таблица устарела, правьте её по коду.

## Ядро рыбалки (`internal/gameserver`)

| Механика | Входные файлы | Единственный вход / ключевое состояние | Тесты |
|---|---|---|---|
| Поклёвка и клёв | `fishing.go`, `fish_hunt.go`, `handlers_game.go` | Корень состояния `FishState`. Цепочка таймеров `scheduleBiteState → onRodBiteTimerTicket → pushPendingBite`. Качество поклёвки `applyHookBiteQuality`, проверка удержания `hookRetentionFails`, вето на «пощипывание» `finishNibbleHook` | `fish_bite_environment_test.go`, `bite_floor_test.go`, `fish_hunt_test.go` |
| Выдача рыбы и уровни | `fishing.go`, `fish_hunt.go`, `fish_distribution.go`, `fish_roll_context.go` | Общая сборка `rollFishForRig → rollFishWithRules`. Четыре уровня `gradeRatioModel`, `sampleWeightByGradeRatio`, `gradeRatioGradeWeights`. Фаза охоты `sampleHuntWeight`. Предел веса `fishWeightCapKg` | `fishing_grade_ratio_test.go`, `fish_distribution_test.go`, `fish_hunt_test.go`, `fishing_weight_cap_test.go` |
| Наживка и пищевые предпочтения | `fishing_bait_rules.go`, `fishing_bait_override.go`, `hook_profiles.go` | Единого входа нет: фильтр предпочтений `filterFishBaitSet` (встроенный `fishing_bait_rules.txt`) / переопределение и ротация `RotationState`, `activeBaitGroup` / тип крючка `hookBiteProfileFor` | `fishing_bait_rules_test.go`, `fishing_bait_override_test.go`, `hook_profiles_test.go` |
| Сборка снасти | `rigops.go`, `lure_hooks.go`, `fishing_rods.go` | Единственный вход записи `rigSetComponent` (семейство команд 13/15/16/17). Положение крючка приманки `lureHookMount`. Состояние нескольких удилищ `fishingStateForRod`. Перестройка проекции `reconcileRigProjection` | `rigops_test.go`, `rigfishing_test.go`, `rig_dup_index_test.go` |
| Нагрузка и поломки | `gear_damage.go`, `gear_physics.go`, `reel_fight_wear.go` | Единого входа нет: поломка детали `breakLineOrRodWithLineOut`, `resolveGearBreak` / списание лески `lineLossTx` / предел нагрузки `gearWeightCapKg`, `gearWeightCapForRig` / износ катушки при вываживании `fightReelCache → settleFightReelWear` | `gear_damage_test.go`, `line_break_loss_test.go`, `reel_fight_wear_test.go` |
| Вываживание и расчёт | `handlers_game.go`, `catch_settlement.go`, `progress_ops.go` | Единственный вход расчёта `settleCatchOnceWithQualification` (квитанция `catchReceipt`). Помещение в садок `fishingKeep`, отпускание `fishingRelease`, смотка `fishingEndSettle` сходятся к нему | `catch_settlement_test.go`, `fishing_contract_test.go`, `fishing_atomic_test.go` |
| Сохранение и восстановление | `fishing_persist.go`, `catch_recovery.go`, `../store/repos_fishing_state.go` | Единственный вход восстановления `resumeFishingState`. Нерассчитанная рыба `recoverCompletedCatchOnLogin` | `fishing_persist_test.go`, `catch_recovery_test.go`, `fish_bite_race_test.go` |

## Системный уровень

| Система | Входные файлы | Единственный вход / ключевой тип | Тесты |
|---|---|---|---|
| Кодирование значений | `../proto/decode.go`, `../proto/encode.go`, `../proto/value.go` | `proto.DecodeValue` / `EncodeValue` (модель значения `proto.Value`) | `../proto/golden_test.go`, `../proto/failure_test.go` |
| Кадры / RC4 / конверт | `../proto/frame.go`, `../proto/rc4.go`, `../proto/envelope.go` | `proto.TakeCompleteFrames`, `proto.NewRC4` (состояние можно перехватить через `Restore`), `proto.ParseEnvelope` | `../proto/frame_test.go` |
| Таблица типов | `../wiregen/tables.go`, `../wiregen/make.go` | `wiregen.Table()`, `wiregen.MakeObj` | `../wiregen/corpus_test.go` |
| Диспетчер команд | `dispatch.go`, `handlers.go` | `NewDispatch` + `Register` / `RegisterFamily`. Регистрация семейств сосредоточена в `registerHandlers` (встроенное переопределение `RegisterExtra`) | `contract_guard_test.go` |
| Выполнение запроса и квитанции | `request_receipts.go`, `../store/repos_request_receipts.go` | Единственный входной шов `(*Server).dispatchRequest`. Ключ идемпотентности `ClaimRequestReceipt` | `request_receipts_test.go` |
| Сессии и пуши | `session.go`, `server.go` | `Session.SendEnvelope` / `Push` / `PushRaw`, `Server.openSession` | `session_sequence_test.go` |
| Шлюз | `../gateway/login.go`, `../gateway/realtime.go`, `../gateway/store.go` | `NewLoginServer`, `NewRealtimeServer`, `gateway.Store` | `../gateway/integration_test.go` |
| Туннель | `../tunnel/tunnel.go`, `../gateway/tunnelclient.go` | `tunnel.NewConn` (формат на линии определяет комментарий в начале файла) | `../tunnel/tunnel_test.go` |
| Хранилище и миграции | `../store/store.go`, `migrations/*.sql` | `store.Open` / `OpenWithStatic`, `(*DB).Migrate` | `../store/migration_stale_branch_test.go`, `data_integrity_test.go` |
| Статическая библиотека только для чтения | `../store/staticdb.go` | `OpenWithStatic` + `staticConnector` (ATTACH на каждое соединение + защита от записи) | `../store/staticdb_test.go`, `../seed/static_runtime_test.go` |
| Сборка профиля | `profilebuild.go` | `BuildProfile307`, `BuildSkillTree311` | `statsbuild_test.go` |
| Уровень и опыт | `progress.go`, `progress_ops.go` | Чистые функции `levelForXp` / `baseXp`. Запись `grantXpMonotonic` | `progress_test.go`, `progress_notifications_test.go` |
| Шкалы состояния (энергия / сытость / здоровье) | `player_condition.go`, `player_condition_energy.go` | Чистая функция `advancePlayerCondition` + `currentPlayerCondition`. Сохранение `UpsertPlayerCondition` | `player_condition_energy_test.go`, `player_condition_health_test.go` |
| Счастливые часы | `happy_hour.go`, `../store/repos_happy_hour.go` | `AdvanceHappyHour` / `GetHappyHourStatus` (переключатель через `tunables.happy_hour_enabled`) | `happy_hour_test.go` |
| Достижения | `achievements_rules.go`, `achievements.go` | `evaluateAchievements` (таблица правил `achievementRule`) | `achievements_rules_test.go`, `achievements_push_test.go` |
| Магазин и ценообразование | `../store/shop_catalog.go`, `../store/shop_augment.go`, `shop.go` | `ShopItems → effectiveShopShelf` (цепочка разбора полок). Протокольная сторона `hStoreBuy` | `../store/shop_catalog_test.go` |
| Рыбный рынок и мастерская | `econ_market.go`, `../store/repos_market_offer.go` | `registerEconMarket` (20/x, 21/x) + чистая функция `marketPriceCents` | `econ_market_test.go`, `../store/repos_market_offer_test.go` |
| Почта / крафт / кафе / ящик с трофеями | `mailbox.go`, `craft_ops.go`, `cafe_orders.go`, `loot_box.go` | `registerMailHandlers`, `hCraft`, `hCafeSubmit`, `rollLootBox` | `mailbox_snapshot_test.go`, `craft_limits_test.go`, `cafe_orders_test.go`, `loot_box_test.go` |
| Админка | `../admin/api.go`, `../admin/bait_rules.go`, `../admin/hook_profiles.go` | Общая сборка маршрутов `admin.NewServer`. `handleBaitRuleSave`, `handleHookProfileSave`. Чтение и запись tunables идёт через `(*DB).TunableSet` | `../admin/api_test.go`, `../admin/bait_rules_test.go`, `../admin/hook_profiles_test.go` |
| Запуск и надзор | `../../cmd/{launcher,gateway,gameserver,seed}/main.go`, `../supervisor/supervisor.go` | Каждый `main()` — единственная точка сборки. `seed.Ensure` | `../supervisor/supervisor_test.go`, `../seed/seed_test.go` |
| Внутриигровой поиск рыбы | `fish_query.go`, `fish_query_water.go`, `fish_query_regions.go` | `(*Server).QueryFish` | `fish_query_test.go`, `fish_query_water_test.go` |

## Структурные наблюдения (в этих местах при чтении кода чаще всего ошибаются)

1. **`FishState` — корень состояния сессии рыбалки**: таймер поклёвки, `Pending`/`Hooked`, метаданные сохранения и книга износа катушки при вываживании висят на одной структуре. Любое изменение нужно регрессировать как «корень состояния целиком», а не только свою механику.
2. **`RolledFish` — межмодульный контракт**: выдача рыбы записывает, поклёвка читает, расчёт потребляет одну и ту же рыбу. Поэтому «поклёвка» и «расчёт» — не две независимые механики, а два конца одной сущности рыбы.
3. **Фактически два ядровых файла**: `fishing.go` (цепочка таймеров + семейство выдачи рыбы + `FishState`/`RolledFish`) и `rigops.go` (всё семейство сборки 13/15/16/17 + регистрация обработчиков). Остальное в основном плагины-стратегии.
4. **В трёх местах единого входа нет намеренно**: наживка (четыре набора: правила / переопределение с ротацией / тип крючка / положение крючка приманки), нагрузка и поломки (пять: поломка / предел нагрузки / износ при вываживании / статический износ / ремонт), подстратегии выдачи рыбы (четыре места: распределение / четыре уровня / охота / предел). Искусственный «общий вход» исказит картину, ищите по подстратегии.
5. **Точка сборки стратегий выдачи рыбы — `fishRollContext`**: `rollFishWithRules` последовательно умножает на один и тот же набор весов распределение, пищевые предпочтения, экологию, погоду и тип крючка. Поэтому выдача рыбы и кажется разбросанной по четырём файлам.
6. **Единственный входной шов — `(*Server).dispatchRequest`**: идемпотентность реализована на уровне диспетчера, а не в обработчиках. Новая механика должна предполагать, что её могут замкнуть квитанцией или остановить защитой от повтора.
7. **Общее состояние собрано в одной структуре `Server`** (таблица сессий + кэши с TTL + несколько мьютексов). `NewServer` — единственный конструктор, тесты тоже идут через него. Это самая большая неявная связность тестов.
8. **`tunables` — единственный канал горячего обновления между эксплуатацией и геймплеем**: админка пишет, store хранит, gameserver читает с TTL. Чтобы проверить горячее обновление, не нужно поднимать весь набор процессов, достаточно вызвать `(*DB).TunableSet`.
9. **Две границы управления**: `admin` держит только `*store.DB` и не вызывает gameserver напрямую (кик и бан идут через очередь команд в БД, gameserver опрашивает и выполняет).
10. **Реализации под названием «отпечаток сборки» нет**: согласованность держится на односторонней перестройке проекции из f5 (`reconcileRigProjection` / `BackfillRigProjections`) + двумерном ключе `rigCompKey(category, ordinal)`. Если поиск `fingerprint` ничего не находит, это нормально.

## Сопровождение

- При переименовании или разбиении входа механики правьте соответствующую строку этой таблицы в том же изменении (в таблице сверки `docs/agents/domain.md` это зарегистрировано).
- Новая механика: сначала добавьте строку в эту таблицу, затем решите, где будет описание поведения: в `docs/gameplay.md` или в `docs/<mechanism>.md`.
- Эта таблица **без номеров строк** и **не копирует правила**. Если строке нужно объяснить «почему», это дело контракта возможностей или ADR.
