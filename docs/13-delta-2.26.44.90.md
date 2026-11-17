# Дельта: 2.26.35.75 → 2.26.44.90 (логин и пост-логин)

Сверка документации (реверс 2.26.35.75, versionCode 263507522) с новой сборкой
**2.26.44.90 (versionCode 264409011)**, снятой с устройства SM-A325F (RZ8RB1LX24T).

## 0. Метод и артефакты

| Что | Где |
|---|---|
| APK 2.26.44.90 | `apk_update/pers_2.26.44.90_base.apk` (147 363 498 Б, sha256 `MSherFwx…`) |
| Декомпиляция новой сборки | `C:\REVERS\wa_params\jadx_264409011\sources` (83 889 классов, jadx 1.5.1; 234 класса не декомпилированы) |
| Старое дерево | `C:\REVERS\wa_params\jadx_out\sources` (2.26.35.75) |
| Нативные библиотеки | `C:\REVERS\wa_params\natives_new\` + `natives_new\decompressed\` (runtime-распаковка superpack) |
| Отчёт по нативке | `natives_new\report_264409011.json` |

Ограничение метода: всё, что помечено **UNKNOWN**, требует живого захвата — тела
методов `ConnectionThread`/`C250617x.A00` jadx не отдал (`Method dump skipped`).

## 1. Итог

| Область | Вердикт |
|---|---|
| Преамбула `57 41 06 03`, edge-заголовок `45 44 00 01` | **OK** (побайтово), но код переехал в state machine `X.2DR.run()` |
| Ветка версии 5 (`57 41 05 03`) | **GONE** — удалена целиком (раньше была недостижима) |
| Framing, `GOA`, лимит 16 MiB | **OK** |
| Noise-паттерны, pqMode, PQ-TLV, ML-KEM-512 | **OK** (имена символ в символ) |
| Сертификат v6 (`BW5`/`BW4`/`Baq`), `WhatsAppLongTerm1` | **OK**; изменён только текст ошибки (+ `issuer=`) |
| Сертификат v5 (`BW6`/`Bar`) и `noise certificate expired` | **GONE** |
| ClientFinish | **OK** |
| **ClientPayload** | **CHANGED**: +2 поля (49 `messageSts`, 50 `integrationAttestation`), версия 2.26.44.90, enum deviceType +2 значения |
| keystore_attestation | **CHANGED**: явный узел `ib/keystore_attestation` (id 6), гейт AB 1935, троттлинг AB 2067 + `ka_stanza_sent_ts` |
| AB 2078/2079 (challenge) | **OK** — читаются через `A0Z` (2078 = длина random-части, 2079 = окно ключа) |
| gpia / safetynet / ACS v1-v2 / edge_routing | **OK** |
| Нативка | **CHANGED**: новый `librust.so` (Shortcake/ML-KEM-768), `libwhatsapp.so` −39 %, `libwhatsappmerged.so` + mldsa65/87 |

## 2. Логин-хендшейк

### 2.1. Преамбула и edge-заголовок — без изменений, но другой код

- `X/C40461qe.A04()` → `{87, 65, 6, 3}` (`jadx_264409011/sources/X/C40461qe.java:288-290`), как и было.
- Порядок записи: `ED 00 01` → 3 байта BE длины `routing_info` → blob → `W A 06 03`.
- Пишет теперь `X.2DR.run()` (state machine), а не конструктор `NoiseSocket`:
  `C2DR.java:33-38` (тело не декомпилировано) → подтверждено дампом из `classes2.dex`.
- Условие edge-заголовка: непустой prefs `routing_info` (base64, лимит 256 Б) — **OK**,
  `C40481qg.java:11-16`, обработчик `<edge_routing>`: `C41461sh.java:134-140`.

### 2.2. Ветка v5 удалена

В 2.26.35.75 в `A04()` была ветка `if (A00 == 6) … else {87,65,5,3}` и лог
`NoiseSocket protocol version is not 5 or 6`. В 2.26.44.90:
- поля `A00` нет, `A04()` — плоская константа;
- grep `NoiseSocket protocol version is not` → **0**;
- нет протобафов v5-сертификата (`BW6`/`Bar`) и строки `noise certificate expired` → **0**.

### 2.3. Framing — OK

- 3 байта BE длины + protobuf `HandshakeMessage`, `flush()` после кадра: `C40511qj.java:20-29`.
- Лимит 16 777 216: инвертировано условие, семантика та же.
- Маркер `GOA {71,79,65}`: `C40501qi.java:9`, сравнение в `C40461qe.java:53-58`.
- `HandshakeMessage` descriptor: `CLIENT_HELLO=2, SERVER_HELLO=3, CLIENT_FINISH=4` — идентичен (`C40881rg.java`).

### 2.4. Noise-паттерны и pqMode — OK

`X/C40601qu.java:16,35-45` содержит те же имена:
`Noise_XX_25519_AESGCM_SHA256`, `Noise_IK_25519_AESGCM_SHA256`,
`Noise_XXfallback_…`, `Noise_XXkem_…`, `Noise_XXkem-FS_…`, `Noise_XXkemEph_…`,
`Noise_IKkem+…`, `Noise_IKkem-FS+…`, `Noise_IKkem2+…`.

- Длинные имена (>32 Б) хешируются SHA-256: `C40601qu.java:68-72`.
- Enum pqMode: `EnumC40841rc.java:8-17` — 0 UNKNOWN, 1 XXKEM, 2 XXKEM_FS, 3 WA_CLASSICAL,
  4 WA_PQ, 5 IKKEM, 6 IKKEM_FS, 7 XXKEM_2, 8 IKKEM_2, 9 XXKEM_EPH — **те же значения**.
- Выбор pqMode и fork full/resume: `C40461qe.A01` (`:79-93`), тот же лог
  `missing serverStaticPQ forcing full handshake`.
- Преамбула микшируется в `h` (`span init_cipher_full=19 / resume=20 / fallback=18`) — **OK**.

### 2.5. Сертификат — OK

- `BW5{leaf=1,intermediate=2}` → `C30042DJo.java`; `BW4{details=1,signature=2}` → `C30043DJp.java`;
  `Baq{serial=1,issuerSerial=2,key=3,notBefore=4,notAfter=5}` → `C30041DJn.java` —
  дескрипторы `newMessageInfo` совпадают со старыми побайтово.
- Корневой ключ `WhatsAppLongTerm1` — те же 32 байта (`AnonymousClass206.java:13-16`).
- Правила проверки (serial/issuerSerial/подписи/сверка key со static) — `C40461qe.java:202-212`.
- Изменение: текст `wa6: noise certificate key does not match proposed server static key`
  теперь с хвостом `; issuer=<issuerSerial>` (`C40461qe.java:260-263`).

### 2.6. ClientFinish — OK

`C40911rj.java`: поля `static=1, payload=2, extendedCiphertext=3, paddedBytes=4,
simulateXxkemFs=5`; поля `signature` нет. Порядок: encrypt_cs (span 10) → ecdh (6) →
encrypt_login_payload (11) → кадр `clientFinish` (поле 4, бит 4) (`C40461qe.java:203-246`).

## 3. ClientPayload

Класс: `X/C39651oO.java` (было `X/C1HS.java`). **38 полей вместо 36.**

### 3.1. Новые поля

| Поле | Номер | Тип | Откуда | Статус |
|---|---|---|---|---|
| `messageSts` | **49** | int64 | контекст `C39631oI.A09` (`duplexMessageStsMicros`), `AbstractC45831zv.java:4732-4739` | NEW |
| `integrationAttestation` | **50** | msg `C8t7{staticUnbound=1, onlineSigned=3}` | Java-сеттеров нет → вероятно нативка | NEW, наполнение UNKNOWN |

Все номера 1–48 сохранены без изменений.

### 3.2. Версия

`appVersion = {2, 26, 44, 90}`, строка `"2.26.44.90"` (`AbstractC39721oq.java:9`,
`AbstractC45831zv.java:4472-4500`); `buildHash` — `Base64.decode(C00M.A04("2.26.44.90"))`
(`AbstractC45831zv.java:4876-4883`).

### 3.3. Уточнения по старым полям

- `lidDbMigrated` (41): условие `A04 != null && !A0A` — в документации было записано `== null` (ошибка доки, не сборки).
- `paaLink` (44): заполнялся и в 2.26.35.75 (кейс DI 2133), в 2.26.44.90 — DI 2212 (`AbstractC29831Tr.java:21555`).
- `deviceType` (userAgent): enum расширен — +`SMART_DISPLAY(5)`, +`SMART_SPEAKER(6)` (`EnumC39811pW.java:8-15`).
- Слот `C39631oI.A09` теперь `Long duplexMessageStsMicros`; `passive` переехал в `A0A`.

### 3.4. Сдвиг DI-номеров (для тех, кто патчит по номерам)

| Что | Было | Стало |
|---|---|---|
| провайдер payload | 6929 | 7376 |
| ключ сортировки | 6930 | 7377 |
| кейсы филлеров | 6556-6561, 6567 | 694-699, 705 |
| paaLink | 2133 | 2212 |

### 3.5. `_lh` и shortcake

- `_lh` — не поле ClientPayload ни в одной версии; Java-кода нет (`grep "_lh"` → 0).
  Код хэшера жив в нативке (`invalid_library_name`, `failed_sha` в `libwhatsapp.so`).
- Кандидат значения для 44.90 (по формуле из `docs/05-integrity/_lh.md`), **требует подтверждения живым захватом**:
  `b64(SHA-256(libwhatsapp.so))` = `4ySx1nCRZmHIIOb5iSSVA0quBT/MIJlyqOQg9FL8gbY=`
  (файл 8 383 928 Б, sha256 `e324b1d6…`); для `libwhatsappmerged.so` —
  `C/GQ+b7wlhHckCq1SsgqG6NLJZm6rHc8T/zLwNtBXOM=`.
- Shortcake (PQ-логин, ML-KEM-768) **переехал в `librust.so`** — см. §5.

## 4. Пост-логин

### 4.1. keystore_attestation — новый путь отправки

- Гейт: `A14(1935)` — тот же AB-флаг (`C250617x.java:203-205`).
- Троттлинг: `A0Z(2067)` + prefs `ka_stanza_sent_ts` (`RunnableC49222Eu.java:89-98`).
- Узел: `new C0T8(new C0T8("keystore_attestation", A04, null), "ib", null)`, id **6**.
- В 2.26.35.75 строкового литерала `keystore_attestation` в Java не было — узел собирался иначе.
- Формат цепочки (root → … → leaf одним blob’ом) — **OK** (`C250617x.java:148-154`).

### 4.2. Challenge — формат тот же, источник может отличаться

`smali1/X/17x.smali:1315-1421`:

- если `p2 == null`: `SecureRandom` (`X/182.A00`) → `new byte[A0Z(2078)]` → `nextBytes`;
- `ByteBuffer.allocate(len + 9)`, `BIG_ENDIAN`, `putLong(clock.A00()/1000)` (8 байт, секунды),
  `put(0x1F)`, `put(random)` → `setAttestationChallenge`.

То есть `challenge = uint64BE(секунды) ‖ 0x1F ‖ random(2078)` — как в 2.26.35.75.

Отличие: при `p2 != null` random не генерируется — используется переданный массив.
В новом пути `2Eu.1.smali:1036` вызывает `17x.A04(..., byte[])`, а байты приходят из
действия `att-stanza-custom`, то есть challenge может быть серверным.

### 4.3. AB 2078/2079 — не удалены

Ранее в отчёте было сказано «удалены» — это ошибка: в 2.26.44.90 они читаются методом
`A0Z`, а не `A0Y`.

| AB | Код | Где | Смысл |
|---|---|---|---|
| 2074 | `0x81a` | `17x.smali:1752,1886` | динамический префикс алиаса |
| 2078 | `0x81e` | `17x.smali:1336` | длина random-части challenge |
| 2079 | `0x81f` | `17x.smali:605,1133` | окно действия ключа |
| 1934 | `0x78e` | `17x.smali:3203` (`A07`) | гейт аттестации |
| 1935 | `0x78f` | `2Eu.1.smali:912` | гейт кастомного узла |
| 2067 | `0x813` | `2Eu.1.smali:972` | троттлинг переотправки |

### 4.4. Без изменений

| Узел | Подтверждение |
|---|---|
| `ib/integrity_payload` (id 194) | `JniBridge.java:887-903` |
| `ib/gpia/jws` (сообщение 254, JNI id 3) | `C16Z.java:16-51` |
| `ib/safetynet/integrity` (сообщение 179) | `C41461sh.java:174-182`, `C39611oG.java:196-204` |
| `edge_routing/routing_info` | `C41461sh.java:134-140` |
| ACS v1 (privatestats, sign_credential v1, blinded_credential 32 Б, id 239) | `JniBridge.java:833-847` |
| ACS v2 (sign_credential v2, project_name/config_id) | `C1XB.java:36-70,163-193` |
| notice/tos/recovery_nonce | `C41461sh.java:193-209` |

### 4.5. Порядок узлов на проводе — UNKNOWN

Все обработчики есть, но разбор `<success>` (`HandlerThreadC08060Yt`) не декомпилирован.
Порядок остаётся по живому захвату 2.26.35.75.

## 5. Нативные библиотеки

| Библиотека | 2.26.35.75 | 2.26.44.90 | Изменение |
|---|---|---|---|
| `libs.so` (superpack) | 13 544 024 | 14 352 736 | +808 712 Б, раскладка та же, архив зашифрован |
| `libsuperpack.so` | 220 920 | 220 920 | **побайтово тот же** |
| `libunwindstack_binary.so` | 11 848 | 11 800 | изменён |
| `libwhatsapp.so` | 14 019 405 (дамп) | 8 383 928 | **−39 %**: Shortcake/ML-KEM-768 ушёл в librust |
| `librust.so` | **нет** | 5 096 368 | **новая**: `messageOne/messageTwoOut/sessionKeyOut/responderState`, `ML-KEM-768`, `shortcake-commitment-v1`, OxidizedCurve25519 |
| `libwhatsappmerged.so` | 8 256 680 | 6 963 952 | `Noise_XX_25519_AESGCM_SHA256` без изменений; + `mldsa65`, `mldsa87`, `Noise context is closed`, labyrinth seal’ы, Tigon MNS/QUIC Noise |

Распакованные runtime-библиотеки: `/data/data/com.whatsapp/files/decompressed/libs.spo/`
(`libwhatsapp.so`, `libwhatsappmerged.so`, `librust.so`, `libcurve25519.so`).

## 6. XMPP / транспорт

- Хост/порт: `g.whatsapp.net`, 443/5222 с историей порта (`C38851mi.java:42-43,61`), id порта 1/4 (`C38781mb.java:45`) — **OK**.
- Framing — бинарный FunXMPP (словарь токенов `AbstractC41781tD.java:11-12`), не XML — **OK**.
- `jabber:client` отсутствует и в новой, и в старой сборке — утверждение доки не подтверждается (не изменение).
- TLS 1.3-обёртка + `WtCachedPsk` — **OK** (`C65318Td2.java:7,33`).

## 7. Что осталось непроверенным

| Пункт | Почему | Как проверить |
|---|---|---|
| Формат ответа `att-stanza-custom` и что приходит в `p2` | тело `LX/1tc`/`LX/27b` не разобрано | Frida-хук на `17x.A04` |
| Поле 50 `integrationAttestation` | нет Java-сеттеров | живой захват ClientPayload + нативка |
| Порядок узлов после `<success>` | `ConnectionThread` не декомпилирован | Frida-хук на `C0T8`/`FunXMPP` |
| `_lh` значение для 44.90 | нативная таблица строк шифрована | дамп памяти/живой `integrity_payload` |
| Точная точка сборки ClientPayload | `HandlerThreadC08060Yt.A0y` не декомпилирован | повторный прогон jadx с `--no-skip` или smali |

## 8. Что делать с инструментом

- Регистрация уже обновлена на 2.26.44.90 (см. `version-config.generated.go`).
- Если реализуется **логин**: в построителе ClientPayload добавить поля 49/50 (номер 49 —
  `duplexMessageStsMicros`, 50 — `integrationAttestation`), версия в `appVersion` — `2.26.44.90`.
- AB-флаги для аттестации: 1935 (вкл.), 2067 (троттлинг), prefs `ka_stanza_sent_ts`; 2078/2079 больше не нужны.
