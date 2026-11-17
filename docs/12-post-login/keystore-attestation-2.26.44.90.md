# keystore_attestation: 2.26.35.75 → 2.26.44.90 (детальный разбор)

English abstract: the attestation *builder* is unchanged in essence (same AB gates 1934/2074/2078/2079,
same challenge layout, same reversed chain order), but the *sending* path is new: a separate
"att-stanza-custom" action gated by AB 1935, throttled by AB 2067 + pref `ka_stanza_sent_ts`,
which emits `<ib><keystore_attestation>` with message id 6.

Источники: `smali1/X/17x.smali` (бывший `C13D` — сборщик ключа/цепочки),
`smali2/X/2Eu.1.smali` (бывший `RunnableC49222Eu` — отправитель).

## 1. Кто собирает и кто отправляет

| | 2.26.35.75 | 2.26.44.90 |
|---|---|---|
| Сборщик | `X/C13D` | `X/17x` (`smali1/X/17x.smali`, 3913 строк) |
| Отправка | внутри логина, DTA case 10 | отдельный путь `X/2Eu`: `att-stanza-custom` → `<ib><keystore_attestation>`, id **6** |
| Гейт | AB 1934 | AB 1934 **и** AB 1935 |
| Троттлинг | нет | AB 2067 + pref `ka_stanza_sent_ts` |
| Alias | AB 2074 | AB 2074 |
| Challenge random | AB 2078 (`A0Y`) | AB 2078 (`A0Z`, `0x81e`) |
| Окно ключа | AB 2079 (`A0Y`) | AB 2079 (`A0Z`, `0x81f`) |

> Важно: AB 2078/2079 **не удалены**. В 2.26.44.90 они читаются методом `A0Z` вместо `A0Y`,
> поэтому grep по `A0Y(2078)` даёт 0. Реальные вхождения: `17x.smali:1336` (`0x81e`),
> `17x.smali:605` и `1133` (`0x81f`).

## 2. Challenge — байт-в-байт та же схема

`17x.A00(Integer,[B)` (`17x.smali:1315-1421`):

```smali
# если p2 == null — генерируем случайную часть
invoke-static {v0}, LX/182;->A00(LX/182;)Ljava/security/SecureRandom;
const/16 v0, 0x81e            # AB 2078 = длина random-части
new-array v9, v0, [B
invoke-virtual {v1, v9}, Ljava/util/Random;->nextBytes([B)V

# сборка challenge
invoke-static/range {v25}, LX/08q;->A00(LX/08q;)J   # скорректированные часы, мс
div-long v0, v0, v16                                 # v16 = 0x3E8 → секунды
invoke-static {v8}, Ljava/nio/ByteBuffer;->allocate(I)  # len(random) + 9
invoke-virtual {v11, v8}, Ljava/nio/ByteBuffer;->order(Ljava/nio/ByteOrder;)  # BIG_ENDIAN
invoke-virtual {v11, v0, v1}, Ljava/nio/ByteBuffer;->putLong(J)               # 8 байт времени
const/16 v0, 0x1f
invoke-virtual {v11, v0}, Ljava/nio/ByteBuffer;->put(B)                        # 0x1F
invoke-virtual {v11, v9}, Ljava/nio/ByteBuffer;->put([B)                       # random
invoke-virtual {v7, v0}, Landroid/security/keystore/KeyGenParameterSpec$Builder;->setAttestationChallenge([B)
```

Итог: `challenge = uint64BE(секунды) ‖ 0x1F ‖ random(A0Z(2078))` — ровно как в 2.26.35.75.

**Новое:** если `p2 != null`, случайная часть **не генерируется** — используется переданный
массив. В новом пути `2Eu.1.smali:1036` вызывает `17x.A04(17x, Integer, byte[])`, где байты
приходят из действия `att-stanza-custom` (`2Eu.1.smali:1018-1036`). То есть в кастомном
узле challenge может быть серверным, а не локальным random.

## 3. Окно действия ключа

`A0Z(2079)` читается дважды — при расчёте `notBefore/notAfter` (`17x.smali:605`) и в
повторном проходе (`17x.smali:1133`). Смысл тот же, что в 2.26.35.75: `now + 2079` секунд.

## 4. Alias

`0x81a` = AB 2074 — динамический префикс алиаса (`17x.smali:1752`, `1886`), как и раньше.
Базовое имя — `att-gen` (`17x.smali:203`), логи `blacknoise/generating key pair with alias=`.

## 5. Цепочка сертификатов

`17x.A03` — тот же обратный обход массива `[leaf … root]` и запись с конца, что и в
`C13D.A03`. Формат: один blob без разделителей, root → … → leaf.

## 6. Новый путь отправки (`2Eu`)

```
17x.A07()                    # AB 1934
  └─ AB 1935 (0x78f)
       └─ pref ka_stanza_sent_ts + A0Z(2067) <= now/1000
            └─ 17x.A04(...)  # сборка цепочки (challenge из att-stanza-custom)
                 ├─ null → Log.w("blacknoise: got null attestation chain in custom stanza")
                 └─ new C0T8("keystore_attestation", chainBytes, null)
                      └─ new C0T8("ib", ...) → send id 6
                      └─ pref ka_stanza_sent_ts = now/1000
```

Строки: `2Eu.1.smali:895-1000` (гейты и троттлинг), `:1036` (сборка), `:1062` (узел),
`:1056-1080` (обёртка `ib` и отправка).

## 7. Что это значит для клиента

- Старый узел в логине не исчез, но появился **отдельный повторяемый** узел
  `ib/keystore_attestation` с id 6, который шлётся не чаще, чем раз в AB 2067 секунд.
- Для повторной отправки нужен серверный обмен `att-stanza-custom` (в нём приходит
  материал challenge) — это новый обязательный шаг, которого не было в 2.26.35.75.
- Формат самого DER-блоба и challenge не изменились — старый код сборки переносится 1:1.

## 8. Осталось непроверенным

| Пункт | Причина | Инструмент |
|---|---|---|
| Точный формат ответа `att-stanza-custom` (`LX/27b`) | тело метода не декомпилировано, smali есть — нужен разбор `LX/1tc`/`LX/27b` | smali/живой захват |
| Что именно приходит в `p2` (challenge от сервера?) | не прослежено до конца | Frida-хук на `17x.A04` |
