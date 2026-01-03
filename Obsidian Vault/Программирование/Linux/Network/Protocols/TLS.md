
```c
typedef enum tls_version {
	SSL_3_0 = 0x0300,
	TLS_1_0 = 0x0301,
	TLS_1_1 = 0x0302,
	TLS_1_2 = 0x0303,
} tls_version_t;

typedef enum tls_content_type {
	CHANGE_CIPHER_SPEC = 20,
	ALERT = 21,
	HANDSHAKE = 22,
	APPLICATION_DATA = 23
} tls_content_type_t;

typedef struct tls_record {
	__u8 content_type;
	__u8 version_major;
	__u8 version_minor;
	__u16 length;
} __attribute__((packed)) tls_record_t;
```

```c
enum tls_handshake_message_type {
	HELLO_REQUEST = 0, // 0 hello_request Запрос на повторное рукопожатие
	CLIENT_HELLO = 1, // 1 client_hello Клиент предлагает параметры соединения
	SERVER_HELLO = 2, // 2 server_hello Сервер выбирает параметры
	NEW_SESSION_TICKET = 4, // 4 new_session_ticket TLS 1.3: новый тикет сессии
	CERTIFICATE = 11, // 11 certificate Сертификат сервера (или клиента)
	KEY_EXCHANGE = 12, // 12 server_key_exchange Ключи для Diffie-Hellman (не в TLS 1.3)
	REQUEST_CERTIFICATE = 13, // 13 certificate_request Запрос сертификата клиента
	SERVER_HELLO_DONE = 14, // 14 server_hello_done Сервер закончил свою часть
	CERTIFICATE_VERIFY = 15, // 15 certificate_verify Подтверждение владения сертификатом
	CLIENT_KEY_EXCHANGE = 16, // 16 client_key_exchange Ключи клиента (для RSA/DHE)
	FINISHED = 20, // 20 finished Подтверждение завершения рукопожатия
} tls_handshake_message_type_t;

typedef struct tls_handshake {
	__u8 msg_type; // offset 0: тип сообщения
	__u8 length[3]; // offset 1-3: длина body (24-bit, big-endian!)
	__u8 body[0]; // variable: TLS handshake body
} __attribute__((packed)) tls_handshake_t;
```


# TLS ClientHello

```c
typedef struct tls_client_hello {
	/* Part 1: Fixed fields */
	__u16 client_version; // offset 0-1: TLS version client supports
	
	/* Part 2: Random (32 bytes) */
	__u8 random[32]; // offset 2-33: 4 bytes timestamp + 28 random
	
	/* Part 3: Session ID */
	__u8 session_id_len; // offset 34: length of session ID
	__u8* session_id; // variable: session ID (if any)
	
	/* Part 4: Cipher Suites */
	__u16 cipher_suites_len; // length in bytes
	__u16* cipher_suites; // variable: list of cipher suites (each 2 bytes)
	
	/* Part 5: Compression Methods */
	__u8 compression_methods_len; // length in bytes
	__u8* compression_methods; // variable: list of compression methods
	
	/* Part 6: Extensions */
	__u16 extensions_len; // length in bytes
	__u8* extensions; // variable: TLS extensions
} tls_client_hello_t;
```

## **Part 1: Fixed fields**

### `client_version` (2 байта)
- **Что содержит**: Версия TLS, которую клиент **поддерживает** (максимальную)
- **Формат**: `0x0303` для TLS 1.2, `0x0304` для TLS 1.3
- **Особенность**: В TLS 1.3 здесь всё равно часто пишут `0x0303` (TLS 1.2) для обратной совместимости, реальная версия указывается в extension `supported_versions`

## **Part 2: Random (32 bytes)**

### `random[32]` (32 байта)
- **Структура**: 
  - Первые **4 байта** (`random[0..3]`) - Unix timestamp (секунды с 1970 года)
  - Остальные **28 байт** (`random[4..31]`) - криптостойкие случайные данные
- **Назначение**: 
  - Участвует в вычислении master secret
  - Предотвращает replay attacks
  - Каждый handshake получает уникальные random значения

## **Part 3: Session ID**

### `session_id_len` (1 байт)
- **Что содержит**: Длина Session ID (0-32 байт)
- **Значение 0**: Клиент не хочет возобновлять сессию

### `session_id` (переменная длина)
- **Что содержит**: Идентификатор сессии (если `session_id_len > 0`)
- **Максимум**: 32 байта
- **Назначение**: 
  - **TLS 1.2 и старше**: Возобновление сессии без полного handshake
  - **TLS 1.3**: Почти не используется, заменён на PSK (Pre-Shared Keys)

**TLS Session** - это криптографический контекст, установленный между клиентом и сервером после успешного handshake. Он включает в себя:

- **Master Secret** (главный секрет, 48 байт)
- **Набор шифров** (cipher suite)
- **Сертификаты** (опционально)
- **Стороны** (client/server random)
- **Session ID** (уникальный идентификатор)

## **Part 4: Cipher Suites**

### `cipher_suites_len` (2 байта)
- **Что содержит**: **Общий размер в байтах** массива `cipher_suites`
- **Внимание**: Это длина в **байтах**, а не количество элементов
- **Всегда чётное число** (каждый cipher suite = 2 байта)

### `cipher_suites` (массив 2-байтовых значений)
- **Что содержит**: Список поддерживаемых криптографических наборов
- **Каждый элемент** - 2 байта (IANA assigned)
- **Порядок**: По убыванию приоритета (клиент предпочитает первый)

**Примеры значений**:
- `0x1301` - TLS_AES_128_GCM_SHA256 (TLS 1.3)
- `0x1302` - TLS_AES_256_GCM_SHA384 (TLS 1.3)
- `0xC02B` - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
- `0xC02F` - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- `0x0005` - TLS_RSA_WITH_RC4_128_SHA (устаревший)

## **Part 5: Compression Methods**

### `compression_methods_len` (1 байт)
- **Что содержит**: Количество методов сжатия (не байты!)
- **Отличие от cipher_suites_len**: Здесь это **количество**, т.к. каждый метод = 1 байт

### `compression_methods` (массив 1-байтовых значений)
- **Что содержит**: Список поддерживаемых алгоритмов сжатия
- **Почти всегда**: `{0x00}` (null compression - без сжатия)
- **Другие значения** (устаревшие):
  - `0x01` - DEFLATE (уязвим к CRIME атаке)
  - `0x40` - LZS

## **Part 6: Extensions**

### `extensions_len` (2 байта)
- **Что содержит**: Общий размер **в байтах** всех расширений
- **Может быть 0** (расширения опциональны)
- **В TLS 1.3**: Обязательно присутствуют (хотя бы supported_versions)

### `extensions` (сырые данные)
- **Что содержит**: Последовательность TLS расширений в формате:
  ```
  [extension_type (2 bytes)] [length (2 bytes)] [data (length bytes)]
  ```

**Важные расширения**:
- **0x0000** - server_name (SNI) - имя хоста
- **0x0005** - status_request (OCSP Stapling)
- **0x000A** - supported_groups (эллиптические кривые)
- **0x000B** - ec_point_formats
- **0x000D** - signature_algorithms
- **0x0010** - application_layer_protocol_negotiation (ALPN)
- **0x002B** - supported_versions (TLS 1.3, критически важно!)
- **0x0033** - pre_shared_key (TLS 1.3 session resumption)
- **0x0039** - key_share (TLS 1.3 key exchange)
- **0x7552** - encrypted_client_hello (ECH)

 **ALPN (Application Layer Protocol Negotiation)** - это механизм, позволяющий клиенту и серверу договориться, какой **прикладной протокол** (HTTP/1.1, HTTP/2, HTTP/3, WebSocket и т.д.) будет использоваться поверх TLS соединения.

# TLS serverHello

```c
typedef struct tls_server_hello {
	/* Part 1: Fixed fields */
	__u16 server_version; // offset 0-1: chosen TLS version
	
	/* Part 2: Random (32 bytes) */
	__u8 random[32]; // offset 2-33: server random
	
	/* Part 3: Session ID */
	__u8 session_id_len; // offset 34: length of session ID
	__u8* session_id; // variable: session ID
	
	/* Part 4: Chosen parameters */
	__u16 cipher_suite; // chosen cipher suite (2 bytes)
	__u8 compression_method; // chosen compression method
	
	/* Part 5: Extensions (optional, TLS 1.3 requires them) */
	__u16 extensions_len; // may be present
	__u8* extensions; // variable: TLS extensions
} __attribute__((packed)) tls_server_hello_t;
```


## Алгоритм обмена Handshak` ами

**Pre-Master Secret** — это 48-байтовая случайная величина, которая **никогда не передаётся в открытом виде**. Она является исходным материалом, из которого обе стороны выведут **Master Secret**. Сам Master Secret уже используется для генерации симметричных ключей шифрования.

### Как происходит обмен в TLS 1.2 (самый наглядный вариант с RSA)

1. **ClientHello** (клиент → сервер)
   - Версия TLS
   - **Client Random** (32 байта)
   - Список поддерживаемых  алгоритмов шифрования в Cipher Suites
   - Расширения (SNI, supported groups и т.д.)

2. **ServerHello** (сервер → клиент)
   - Выбранная версия TLS
   - **Server Random** (32 байта)
   - Выбранный алгоритм шифрования Cipher Suite
   - Расширения

3. **Сервер отправляет свой сертификат** (содержит публичный ключ RSA)

4. **Клиент генерирует Pre-Master Secret** (48 случайных байт) **у себя локально**

5. **Клиент шифрует этот Pre-Master Secret** **публичным ключом RSA** из сертификата сервера и отправляет в сообщении **ClientKeyExchange**

6. **Сервер расшифровывает Pre-Master Secret** своим приватным ключом RSA

7. **Теперь у обоих есть:**
   - Client Random (из ClientHello)
   - Server Random (из ServerHello)
   - Pre-Master Secret (только что переданный зашифрованным)

8. **Обе стороны независимо вычисляют Master Secret** по формуле:
   ```
   master_secret = PRF(pre_master_secret, "master secret",
                       client_random + server_random)
   ```

9. **Из Master Secret выводятся ключи симметричного шифрования** (AES keys, HMAC keys)

10. **Далее общаются симметрично**

### В случае с ECDHE (самый распространённый сейчас)

- Вместо RSA-шифрования pre-master secret используется **алгоритм Диффи-Хеллмана на эллиптических кривых**.
- Клиент и сервер обмениваются публичными ключами (уже в ClientHello / ServerHello через расширение `key_share`).
- **Pre-Master Secret вычисляется** как результат ECDHE-обмена (общий секрет).
- Этот общий секрет используется вместо pre-master secret для вычисления master secret.
- **Никакого шифрования pre-master secret не происходит** — он вычисляется, а не передаётся.

### Краткая схема (RSA-вариант)

```
Клиент                            Сервер
  |--- ClientHello (random1) ----->|
  |<--- ServerHello (random2) + сертификат
  |                                 |
  | (генерирует pre-master secret)  |
  | (шифрует его публичным ключом)  |
  |--- ClientKeyExchange (зашифр.) ->| (расшифровывает)
  |                                 |
  |<--- ChangeCipherSpec + Finished | (переход на симметричное шифрование)
  |--- ChangeCipherSpec + Finished ->|
  |<--- зашифрованные данные ------->|
```

В ECDHE-варианте `ClientKeyExchange` заменяется на обмен ключами в `key_share`, и pre-master secret вычисляется, а не передаётся.

---

Может на первый взгляд показаться, что можно обойтись только `pre-master secret`, зашифрованным публичным ключом сервера. Но `client_random` и `server_random` нужны для **уникальности ключа сессии** и защиты от **replay-атак**.

### 1. Уникальность ключа для каждой сессии

Даже если клиент использует один и тот же `pre-master secret` (например, в случае повторного соединения или из-за ошибки в генераторе случайных чисел), добавление разных `client_random` и `server_random` гарантирует, что итоговый `master_secret` будет разным. Формула:
```
master_secret = PRF(pre_master_secret, "master secret", client_random + server_random)
```
- Если `client_random` и `server_random` уникальны для каждого соединения (а они генерируются заново каждый раз), то `master_secret` будет уникальным, даже если `pre_master_secret` совпадёт.

### 2. Защита от replay-атак

Злоумышленник может записать старый `ClientHello` и `ClientKeyExchange` (где зашифрован `pre_master_secret`) и попытаться воспроизвести их позже. Если бы не было случайных значений, сервер при воспроизведении получил бы тот же `pre_master_secret` и сгенерировал бы те же ключи. Это позволило бы злоумышленнику расшифровать перехваченный трафик (если он также записал его). Но благодаря `client_random` и `server_random`, которые сервер выбирает случайно (и они не повторяются), даже если злоумышленник воспроизведёт старый `ClientHello` и `ClientKeyExchange`, сервер ответит новым `ServerHello` со своим случайным значением, и итоговый `master_secret` будет другим. Старый перехваченный трафик не удастся расшифровать.

### 3. Доказательство свежести (freshness)

Случайные значения гарантируют, что рукопожатие происходит «здесь и сейчас». Это стандартный механизм в криптографических протоколах — **nonce**.

### 4. Безопасность PRF

Функция псевдослучайного вывода (PRF) в TLS использует все три компонента. Если бы `pre_master_secret` был скомпрометирован (например, из-за уязвимости генератора случайных чисел), случайные значения всё равно добавляют энтропию, усложняя задачу злоумышленнику.

### 5. Совместимость с разными алгоритмами обмена ключами

В TLS 1.2 `pre_master_secret` может вычисляться по-разному (RSA-шифрование, ECDHE, PSK). Добавление случайных значений унифицирует вывод `master_secret` для всех методов.

### Коротко: если бы не было `client_random` и `server_random`

- Одна и та же пара (клиент, сервер) всегда использовала бы одни и те же ключи при повторном соединении (если `pre_master_secret` не меняется).
- Это позволило бы злоумышленнику воспроизводить старые рукопожатия.
- Кроме того, любой, кто узнал `pre_master_secret` (например, через взлом сервера), смог бы расшифровать **все прошлые сессии**, где использовался этот секрет, даже не зная случайных значений? На самом нет, потому что `pre_master_secret` обычно уникален для сессии. Но если бы он был одинаков для многих сессий, то случайные значения обеспечивали бы различие.
