
**Web Application Firewall (WAF) — это специализированный межсетевой экран (firewall) уровня приложений (L7 по модели OSI), предназначенный для защиты веб-приложений от атак, направленных на эксплуатацию уязвимостей в HTTP/HTTPS-трафике и бизнес-логике приложения.**

---

##  1. **Основная цель WAF**
Обнаружение, фильтрация и блокировка вредоносных HTTP/HTTPS-запросов **до** того, как они достигнут веб-приложения, при этом сохраняя легитимный трафик.

В отличие от сетевых фаерволов (L2/L3/L4), WAF:
- работает с семантикой HTTP(S) — заголовки, методы (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `HEAD` и др.), тела запросов и ответов, URI, куки, параметры;
- понимает структуру данных: `application/x-www-form-urlencoded`, `multipart/form-data`, `application/json`, `application/xml`, `text/plain`, GraphQL и др.;
- может глубоко анализировать содержимое и поведение.

---

## 2. **Основные функции и возможности WAF**

### 🛡️ **2.1. Обнаружение и предотвращение атак (Intrusion Prevention)**

| Угроза / атака                         | Примеры                                                                                 | Как WAF это обнаруживает/блокирует                                                                                           |
| -------------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **SQL Injection (SQLi)**               | `' OR 1=1--`, `UNION SELECT`, `SLEEP()`, `BENCHMARK()`                                  | Сигнатурный анализ, семантический разбор SQL-подобных паттернов, anomaly detection, ML                                       |
| **Cross-Site Scripting (XSS)**         | `<script>alert(1)</script>`, `javascript:`, `onerror=`, `&#x6A;&#x61;...`               | Анализ HTML-контекста, проверка на `<`, `>`, `&`, `"`/`'`, проверка escape-последовательностей, контекстно-зависимые правила |
| **Command Injection**                  | `; ls`, `                                                                               | cat /etc/passwd`, `$(id)`, backticks                                                                                         |
| **Path Traversal**                     | `../../../etc/passwd`, `%2e%2e%2f`, `..%5c`, `C:\boot.ini`                              | Нормализация URI, проверка на `..`, `\`, `/`, нестандартные кодировки                                                        |
| **Server-Side Request Forgery (SSRF)** | `http://127.0.0.1:8080`, `http://localhost`, `file:///`, `dict://`, `gopher://`         | Блокировка запрещённых доменов/IP/протоколов, allow-list внешних endpoint’ов                                                 |
| **XML External Entity (XXE)**          | `<!ENTITY xxe SYSTEM "file:///etc/passwd">`                                             | Анализ `DOCTYPE`, блокировка внешних entity, отключение DTD-парсинга                                                         |
| **Remote Code Execution (RCE)**        | `eval($_POST['code'])`, `pickle.loads()`, `pickle` в Python, `ObjectInputStream` в Java | Сочетание сигнатур, аномалий, контроля входных данных в чувствительных параметрах                                            |
| **Local File Inclusion (LFI)**         | `?page=../../etc/passwd`, `php://filter/read=convert.base64-encode/resource=index.php`  | Анализ значений параметров, блокировка wrapper’ов (`php://`, `data://`, `zip://`)                                            |
| **HTTP Request Smuggling**             | `Transfer-Encoding: chunked`, `Content-Length` десинхронизация, `TE: trailers`          | Строгая валидация заголовков, нормализация, единый HTTP-парсер на всех уровнях                                               |
| **GraphQL-specific угрозы**            | Запросы с deep nesting, интенсивные интроспекции, `__schema`, `__typename`              | Лимит глубины запроса, disable интроспекции, cost-based analysis                                                             |
| **JSON Injection / NoSQL Injection**   | `{"$ne": ""}`, `{"$regex": ".*"}`, `{"$where": "..."}`                                  | Проверка на MongoDB/Redis/NoSQL-операторы в JSON                                                                             |
| **CSRF (частично)**                    | Отсутствие/невалидный `Origin`, `Referer`, `X-Requested-With`, `CSRF-Token`             | Проверка CORS-заголовков, enforce `SameSite=Strict` для кук, token-binding                                                   |
| **HTTP Parameter Pollution (HPP)**     | `?id=1&id=2&id=3`, повторяющиеся параметры в разных кодировках                          | Нормализация и контроль уникальности/количества параметров                                                                   |
| **Open Redirect**                      | `?next=http://evil.com`, `?url=data:text/html,...`                                      | Allow/deny list целей перенаправления, валидация URI                                                                         |

#### **1. SQL Injection (SQLi)**
*   **Суть атаки:** Внедрение злонамеренного SQL-кода в параметры запроса (GET/POST, cookies, заголовки) для чтения/изменения/удаления данных в базе или выполнения команд на сервере БД.
*   **Примеры:**
    *   `' OR 1=1--` – Классическая "тавтология", обходит проверку пароля.
    *   `UNION SELECT username, password FROM users--` – Извлекает данные из других таблиц.
    *   `SLEEP(5)`, `BENCHMARK(1000000, MD5('test'))` – "Слепые" SQLi для определения уязвимости и подбора данных по времени отклика.
*   **Как WAF обнаруживает/блокирует:**
    *   **Сигнатурный анализ (черные списки):** Ищет известные SQL-ключевые слова (`UNION`, `SELECT`, `DROP`, `EXEC`, `--`, `#`, `'`, `"`), функции (`SLEEP`, `BENCHMARK`, `WAITFOR`), и их обфусцированные версии (например, `SEL/**/ECT`).
    *   **Семантический разбор (синтаксический анализ):** Пытается понять структуру потенциального SQL-запроса в данных пользователя. Распознает не просто слова, а *конструкции* (например, последовательность `'` + `OR` + условие + `--`).
    *   **Anomaly detection (анализ аномалий, белые списки):** Создает базовый профиль "нормальных" параметров для каждого endpoint (длина, тип символов, структура). Резкое отклонение (например, параметр `id`, который обычно содержит 1-3 цифры, внезапно содержит 200 символов со знаками препинания) вызывает подозрение.
    *   **Машинное обучение (ML):** Расширенная форма anomaly detection. Модель обучается на огромных объемах легитимного и вредоносного трафика, выявляя сложные, неявные паттерны SQLi, которые трудно описать правилами.
    *   **Нормализация/декодирование:** Перед проверкой WAF декодирует URL-encoded (`%20` -> пробел), Unicode, двойное кодирование, шестнадцатеричные представления, чтобы атака не прошла в замаскированном виде.
*   **Современный тренд:** Комбинация методов. Сначала быстрые сигнатуры, затем семантический анализ и ML-модели для сложных случаев. Ключевая задача – минимизировать ложные срабатывания (False Positives).

#### **2. Cross-Site Scripting (XSS)**
*   **Суть атаки:** Внедрение вредоносного JavaScript-кода в веб-страницу, который выполняется в браузере жертвы. Бывает: **Reflected** (код в URL, отражается сразу), **Stored** (код сохраняется на сервере, например, в комментариях), **DOM-based** (обрабатывается полностью на стороне клиента).
*   **Примеры:**
    *   `<script>alert('XSS')</script>` – Классика.
    *   `javascript:alert(1)` – Внутри атрибутов (`href`, `src`).
    *   `onerror=alert(1)` – Использование событий (event handlers) HTML.
    *   `&#x6A;&#x61;&#x76;&#x61;...` – HTML-entities для обфускации.
*   **Как WAF обнаруживает/блокирует:**
    *   **Анализ HTML-контекста (Context-Aware Security):** Самый важный и сложный аспект. WAF пытается определить, куда попадут данные пользователя:
        *   **HTML Body (`<div> [пользовательские_данные] </div>`):** Блокирует незакрытые теги, `<script>`, `<iframe>`, `<object>`.
        *   **HTML Attribute (`<input value="[пользовательские_данные]">`):** Блокирует кавычки, которые позволят "выйти" из атрибута, и конструкции `on*` (onclick, onerror).
        *   **JavaScript (`<script>var a = "[пользовательские_данные]";</script>`):** Блокирует закрывающие кавычки, точки с запятой, скобки, которые позволят внедрить новый JS-код.
        *   **URL (`<a href="[пользовательские_данные]">`):** Проверяет протокол, блокирует `javascript:`.
    *   **Проверка на escape-последовательности:** Распознает и декодирует обфускации: HTML-entities (`&#x3C;`), JS-unicode (`\u003c`), кодировки Base64 в `data:` URI.
    *   **Политики безопасности контента (CSP) – поддержка:** WAF может не только блокировать, но и генерировать или проверять заголовок `Content-Security-Policy`, который инструктирует браузер, откуда разрешено загружать скрипты и стили.

#### **3. Command Injection**
*   **Суть атаки:** Попытка выполнить произвольные операционные команды на сервере через уязвимые параметры (например, параметр `ip` в команде `ping`).
*   **Примеры:**
    *   `; ls` – В Unix `;` разделяет команды.
    *   `| cat /etc/passwd` – `|` передает вывод одной команды на вход другой.
    *   `$(id)`, `` `id` `` – Подстановка команд в shell.
    *   `& dir` – В Windows `&` разделяет команды.
*   **Как WAF обнаруживает/блокирует:**
    *   **Сигнатуры метасимволов shell:** Поиск символов и строк, которые имеют особое значение в командных оболочках (`;`, `|`, `&`, `$()`, `` ` ``, `\n`, `&&`, `||`).
    *   **Сигнатуры команд ОС:** Поиск имен команд (`ls`, `cat`, `dir`, `whoami`, `netstat`, `nc`, `wget`, `curl`).
    *   **Сигнатуры путей и файлов:** Поиск чувствительных путей (`/etc/passwd`, `/bin/sh`, `C:\Windows\System32\cmd.exe`).
    *   **Anomaly detection:** Анализ типичных значений для параметра. Если поле `username` никогда не содержало пробелов или точек с запятой, а теперь содержит `; rm -rf /`, это явная аномалия.

#### **4. Path Traversal (Directory Traversal)**
*   **Суть атаки:** Использование последовательностей `..` (родительский каталог) для доступа к файлам за пределами корневой директории веб-приложения.
*   **Примеры:**
    *   `../../../etc/passwd` – Классика.
    *   `..%2f` или `%2e%2e%2f` – URL-encoded версия (`/` -> `%2f`, `.` -> `%2e`).
    *   `..%5c` – Обратный слеш для Windows (`\` -> `%5c`).
    *   `C:\boot.ini` – Прямой доступ к файлам в Windows.
*   **Как WAF обнаруживает/блокирует:**
    *   **Нормализация URI:** Приведение пути к каноническому виду. Декодирует все кодировки, заменяет `\` на `/`, убирает избыточные слеши.
    *   **Поиск паттернов обхода:** После нормализации ищет последовательности `../` (и `..\`). Даже одна такая последовательность в параметре файла (`?file=../../../`) может быть подозрительной.
    *   **Проверка на абсолютные пути:** Блокировка путей, начинающихся с `C:\`, `/etc/`, `/root/` и т.д.
    *   **Белые списки (предпочтительный метод):** WAF может проверять, что итоговый путь после обработки находится в разрешенной директории (например, в `./public_html/`).

#### **5. Server-Side Request Forgery (SSRF)**
*   **Суть атаки:** Заставить сервер сделать произвольный HTTP-запрос к внутренним (loopback) или сторонним ресурсам, которые недоступны напрямую атакующему.
*   **Примеры:**
    *   `http://127.0.0.1:8080/admin` – Обращение к внутреннему административному интерфейсу.
    *   `http://localhost` – Альтернативное обращение к loopback.
    *   `file:///etc/passwd` – Чтение локальных файлов через протокол `file://`.
    *   `dict://localhost:11211/stats` – Атака на сервисы, использующие текстовые протоколы (например, Memcached).
*   **Как WAF обнаруживает/блокирует:**
    *   **Deny-list запрещенных целей:** Блокировка IP-адресов (127.0.0.1, 0.0.0.0, 169.254.169.254 – метаданные облаков), доменов (`localhost`, `internal`), и подсетей (например, всей частной сети 10.0.0.0/8).
    *   **Deny-list запрещенных протоколов:** Блокировка схем `file://`, `gopher://`, `dict://`, `ftp://`.
    *   **Allow-list разрешенных ресурсов (рекомендуется):** Разрешение запросов только к заранее одобренному списку доменов (например, `cdn.example.com`, `api.payment.com`).
    *   **DNS Rebinding защита:** WAF может выполнять разрешение доменного имени и проверять, не ведет ли оно на запрещенный IP (например, домен может сначала вернуть "безопасный" IP, а при повторном запросе – внутренний).

#### **6. XML External Entity (XXE)**
*   **Суть атаки:** Использование уязвимого XML-парсера, который обрабатывает внешние сущности (External Entities), для чтения локальных файлов, сканирования сети или выполнения запросов.
*   **Пример:**
    *   `<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]> <foo>&xxe;</foo>` – Определение внешней сущности, которая будет заменена содержимым файла.
*   **Как WAF обнаруживает/блокирует:**
    *   **Анализ DTD:** Поиск в теле запроса (не только в заголовках) ключевых слов `<!DOCTYPE`, `<!ENTITY`, `SYSTEM`, `PUBLIC`.
    *   **Блокировка внешних сущностей:** Даже если DTD обнаружен, WAF может заблокировать обработку сущностей с атрибутом `SYSTEM`.
    *   **Блокировка файловых схем:** В рамках защиты от XXE также блокируются строки типа `file:///`, `php://filter/`.
    *   **Принудительная настройка парсера:** WAF может добавить заголовки или модифицировать запрос, чтобы указать серверному приложению использовать безопасный парсер с отключенными DTD и внешними сущностями.

#### **7. Remote Code Execution (RCE)**
*   **Суть атаки:** Наиболее критичная уязвимость, приводящая к выполнению произвольного кода на сервере.
*   **Примеры:**
    *   `eval($_POST['code'])` – Прямое выполнение кода из PHP-запроса.
    *   `pickle.loads(USER_INPUT)` – Уязвимая десериализация в Python.
    *   `ObjectInputStream.readObject()` – Уязвимая десериализация в Java.
*   **Как WAF обнаруживает/блокирует:**
    *   **Сочетание сигнатур:** Поиск названий опасных функций (`eval`, `exec`, `system`, `popen`, `unserialize`, `pickle.loads`).
    *   **Анализ сериализованных данных:** Для Java/C#/Python WAF может иметь парсеры, которые проверяют структуру сериализованных объектов на наличие подозрительных классов или гаджетов (gadget chains), ведущих к RCE.
    *   **Контроль входных данных в "горячих" параметрах:** Параметры с именами вроде `command`, `code`, `data`, `input` проверяются с максимальной строгостью, часто по белым спискам.
    *   **Anomaly detection и ML:** Резкое увеличение длины или сложности параметра, который обычно прост, может указывать на попытку внедрения кода.

#### **8. Local File Inclusion (LFI)**
*   **Суть атаки:** Включение локальных файлов в вывод веб-страницы через уязвимые параметры (часто `?page=about.php`).
*   **Примеры:**
    *   `?page=../../etc/passwd` – Классический LFI, пересекается с Path Traversal.
    *   `?page=php://filter/convert.base64-encode/resource=index.php` – Использование PHP wrapper'а для чтения исходного кода в base64, минуя его выполнение.
*   **Как WAF обнаруживает/блокирует:**
    *   **Анализ значений параметров:** Поиск Path Traversal-последовательностей.
    *   **Блокировка wrapper'ов:** Поиск и блокировка строк `php://`, `data://`, `zip://`, `expect://`, `phar://` в параметрах файлов.
    *   **Валидация расширений (не всегда надежно):** Проверка, что конечный файл имеет ожидаемое расширение (например, `.php`, `.html`), хотя это можно обойти с помощью null-byte (`%00`) в старых системах.

#### **9. HTTP Request Smuggling**
*   **Суть атаки:** Эксплуатация разницы в обработке HTTP-запросов между фронтендом (WAF/балансировщик) и бэкендом (сервер приложения) для "проталкивания" скрытого злонамеренного запроса.
*   **Примеры:**
    *   Несогласованность `Content-Length` и `Transfer-Encoding: chunked`.
    *   Использование устаревших или нестандартных особенностей HTTP (`HTTP/1.0`, `Keep-Alive`, заголовки `X-...`).
*   **Как WAF обнаруживает/блокирует:**
    *   **Строгая валидация заголовков:** Недопущение противоречивых (`Content-Length` и `TE: chunked`) или нестандартных заголовков.
    *   **Нормализация запроса:** WAF должен полностью перепарсить входящий запрос, очистить его от неоднозначностей и отправить на бэкенд в стандартизированном виде.
    *   **Единый HTTP-парсер:** Использование одного и того же надежного парсера на всех точках инфраструктуры (WAF, балансировщик, сервер) – самый эффективный метод предотвращения.

#### **10. GraphQL-specific угрозы**
*   **Суть атаки:** Эксплуатация специфических особенностей GraphQL: глубоко вложенные запросы, интроспекция (запрос схемы), отсутствие лимитов по сложности.
*   **Примеры:**
    *   **Deep Nesting:** `{ user { posts { comments { author { posts ... } } } } }` – Циклический запрос, который может вызвать огромную нагрузку (DoS).
    *   **Интроспекция:** Запрос `{ __schema { types { name fields { name } } } }` для получения полной карты API и поиска скрытых или тестовых полей.
*   **Как WAF обнаруживает/блокирует:**
    *   **Лимит глубины запроса (max depth):** Запрет запросов с уровнем вложенности выше допустимого (например, больше 5).
    *   **Лимит сложности (max complexity):** Присвоение "веса" каждому полю и лимит на общую сложность запроса.
    *   **Отключение интроспекции в prod-среде:** WAF может блокировать запросы, содержащие поля `__schema`, `__type`, если это не разрешено.
    *   **Persisted Queries:** WAF может проверять, что приходит не произвольный GraphQL-запрос, а только его хэш (идентификатор) из заранее одобренного списка.

#### **11. JSON Injection / NoSQL Injection**
*   **Суть атаки:** Аналогична SQLi, но для NoSQL баз данных (MongoDB, Redis, CouchDB, Cassandra). Внедрение операторов запросов через JSON или другие форматы.
*   **Примеры (MongoDB):**
    *   `{"$ne": ""}` – Оператор "not equal" для обхода проверки логина.
    *   `{"$regex": ".*"}` – Использование регулярного выражения.
    *   `{"$where": "this.password == 'secret'"}` – Выполнение JS-кода на сервере MongoDB.
*   **Как WAF обнаруживает/блокирует:**
    *   **Проверка на операторы NoSQL:** Поиск в JSON-телах запросов префиксов `$` (`$ne`, `$regex`, `$where`, `$gt`, `$or`) и их обфусцированных версий.
    *   **Проверка на логические операторы:** Поиск ключевых слов `OR 1=1` в строковом виде внутри JSON.
    *   **Валидация схемы JSON:** Если ожидается простой объект `{"username": "string"}`, попытка передать `{"username": {"$ne": null}}` будет заблокирована.

#### **12. CSRF (частично)**
*   **Суть атаки:** Cross-Site Request Forgery – вынуждение браузера жертвы отправить авторизованный запрос на уязвимое веб-приложение без его ведома.
*   **Как WAF помогает (пассивная/активная роль):**
    *   **Проверка заголовков Origin/Referer:** WAF может проверять, что для запросов, изменяющих состояние (POST, PUT, DELETE), заголовки `Origin` или `Referer` соответствуют домену приложения, блокируя межсайтовые запросы. **Важно:** Это не абсолютно надежно (заголовки могут отсутствовать по легитимным причинам, например, при переходе с HTTPS на HTTP).
    *   **Проверка наличия CSRF-токена:** WAF может проверять, что в телах запросов или заголовках присутствует специальный токен (например, `X-CSRF-Token`), и даже валидировать его, если имеет доступ к сессии пользователя (режим "reverse proxy").
    *   **Принудительная политика SameSite для кук:** WAF может модифицировать устанавливаемые сервером cookie, добавляя к ним атрибут `SameSite=Strict` или `Lax`, что предотвращает их отправку в межсайтовых запросах.

#### **13. HTTP Parameter Pollution (HPP)**
*   **Суть атаки:** Передача нескольких параметров с одинаковыми именами (`?id=1&id=2`). Разные технологии (PHP, ASP.NET, JSP) обрабатывают это по-разному, что может привести к логическим ошибкам или обходу валидации.
*   **Примеры:**
    *   `?id=1&id=2` – PHP прочитает последнее значение (`id=2`), JSP может прочитать массив.
    *   `?id=1%26id=2` – Двойное кодирование.
*   **Как WAF обнаруживает/блокирует:**
    *   **Нормализация и контроль уникальности:** WAF может приводить параметры к каноническому виду и либо выбирать первое/последнее значение, либо блокировать запрос, если обнаружены дубликаты (в зависимости от политики).
    *   **Лимит количества параметров:** Запрос с аномально большим числом параметров может быть заблокирован.

#### **14. Open Redirect**
*   **Суть атаки:** Использование легитимной функции перенаправления на сайте для отправки пользователя на фишинговый или вредоносный ресурс.
*   **Примеры:**
    *   `?next=http://evil-phishing.com`
    *   `?url=data:text/html,<script>alert('phish')</script>` – Перенаправление на вредоносную data-URI.
*   **Как WAF обнаруживает/блокирует:**
    *   **Allow/Deny list целей:** Разрешение перенаправлений только на доверенные домены (относительные пути, свой домен, партнерские домены).
    *   **Валидация URI:** Проверка схемы (`http://`, `https://`, `data://`, `javascript:`), домена и пути. Блокировка внешних URL, если функция предназначена только для внутренних переходов.
    *   **Токенизация URL:** Использование не прямого URL, а токена/идентификатора, который сервер сопоставляет с разрешенным URL.


---

### **2.2. Обнаружение аномалий и поведенческий анализ**

| Функция | Описание |
|--------|---------|
| **Rate limiting / Throttling** | Ограничение запросов по IP, User-Agent, Session-ID, API-key (например: 100 req/min/IP/user) |
| **Brute-force protection** | Обнаружение множества неудачных логинов, OTP-попыток (по `login`, `2fa_code`, `password`) |
| **Bot Mitigation** | Анализ заголовков (`User-Agent`, `Accept`, `Sec-CH-UA`), проверка JS-челленджей (например, Cloudflare Turnstile), TLS/JA3 fingerprinting, CAPTCHA/JS-challenge, headless browser detection |
| **Anomaly-based detection** | ML-модели: отклонение от baseline по длине URI, кол-ву параметров, entropy заголовков, частоте методов |
| **Session integrity checks** | Контроль `Cookie`, `Authorization`, `X-Session-ID` — несоответствия между регионом, User-Agent, IP |
| **Reputation-based blocking** | IP из блэклистов (Spamhaus, AbuseIPDB), известные TOR-выходы, сканеры (nmap, sqlmap UA), мальварные AS |

---

### **2.3. Защита данных и конфиденциальности**

| Возможность | Примеры |
|-------------|---------|
| **Data Loss Prevention (DLP)** | Поиск и маскировка/блокировка утечки: ИНН, номера карт (`4\d{3} ?\d{4} ?\d{4} ?\d{4}`), паспортных данных, email’ов в теле ответа |
| **PCI DSS compliance** | Обязательные правила для обработки платёжных данных (ограничение логгирования PAN, CVV) |
| **PII masking** | Подмена полей в ответе: `"email": "user@example.com"` → `"email": "u***@e***.com"` |
| **Cookie security enforcement** | Автоматическое добавление `Secure`, `HttpOnly`, `SameSite=Strict/Lax` |

---

### **2.4. Управление трафиком и маршрутизация**

| Функция | Применение |
|--------|-----------|
| **Virtual Patching** | "Заплатка" без изменения кода: блокировка уязвимости, пока команда не выпустит фикс (например, CVE в WordPress) |
| **Request/Response manipulation** | Удаление `Server: Apache/2.4.41`, `X-Powered-By`, замена `Location` при редиректах, добавление `Content-Security-Policy`, `X-Frame-Options` |
| **API Gateway функции** | Валидация OpenAPI/Swagger-схемы, enforcement методов/путей, JWT introspection/verification, OAuth2 scopes |
| **A/B testing & feature flags (иногда)** | На основе заголовков (`X-Experiment: v2`) — маршрутизация в разные бэкенды |

---

### **2.5. Мониторинг, логгирование и интеграция**

| Компонент | Поддержка |
|----------|----------|
| **Logging** | Запись blocked/allowed запросов (включая `request_id`, тело, IP, правило, действие) — в JSON, syslog, Kafka, ClickHouse |
| **Alerting** | Интеграция с SIEM (ELK, Splunk, Graylog), Slack, PagerDuty, Prometheus/Alertmanager |
| **Metrics** | Графики: req/sec, блокировки/час, top-атаки, latency, ложные срабатывания (FP rate) |
| **Forensic data** | Full packet capture (PCAP) по событию, тело запроса/ответа (при нарушении политики) |
| **Integration** | REST API для управления, Terraform provider, Ansible modules, OpenTelemetry экспортеры |

---

### **2.6. Режимы работы**

| Режим | Описание | Плюсы / Минусы |
|------|----------|----------------|
| **Blocking (Enforcement)** | Запросы блокируются (`403`, `406`, `444`, `503`) | ✅ Защита. ❌ Риск FP (ложных срабатываний) → downtime |
| **Detection-only (Monitoring)** | Запросы пропускаются, но логируются и алерты генерируются | ✅ Безопасное внедрение. ❌ Нет защиты до перехода в блокировку |
| **Hybrid (Challenge + Blocking)** | Подозрительный трафик → CAPTCHA/JS-челлендж → если пройден — пропускается | ✅ Баланс UX и безопасности. ❌ Требует JS, может нарушать SEO/API |

---

## 3. **Архитектурные варианты WAF**

| Тип                               | Примеры                                                                                      | Особенности                                                                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Cloud-based (SaaS)**            | Cloudflare WAF, AWS WAF, Akamai App Protect, Imperva Incapsula                               | 🌐 Легко включить, DDoS-защита «из коробки», global footprint. ❌ Зависимость от провайдера, латентность, limited custom logic |
| **Reverse Proxy (On-prem/Cloud)** | ModSecurity + Nginx/Apache, F5 ASM, Barracuda WAF                                            | ⚙️ Полный контроль, можно ставить перед балансировщиком. Требует ресурсов и экспертизы                                        |
| **In-App / Embedded**             | ModSecurity + libmodsecurity3 (Standalone), Coraza (Go), Sqreen (deprecated), Datadog AppSec | 🐍 Интеграция в runtime (Python/Java/Node.js). Минимальная latency, но — overhead в приложении                                |
| **Service Mesh (Sidecar)**        | Envoy + ModSecurity filter, Tetrate WAF, Solo.io Gloo Edge                                   | 🧱 Идеально для microservices. WAF в Istio sidecar. Высокая гибкость и observability                                          |
| eBPF/XDP                          |                                                                                              |                                                                                                                               |

---

## 4. **Современные продвинутые возможности**

| Фича | Описание |
|------|---------|
| **Runtime Application Self-Protection (RASP) интеграция** | WAF получает контекст из приложения: stack trace, переменные, SQL-запросы → высокоточная блокировка (Wallarm, Contrast Security) |
| **Threat Intelligence Feeds** | Автоматическая подписка на IOCs (IP, домены, хэши), обновление блэклистов раз в 5–15 мин |
| **Decoy / Honeypot pages** | `/admin.php.bak`, `/phpinfo.php` → любое обращение = бот/атака → блок IP |
| **GraphQL-aware protection** | Разбор AST, подсчёт complexity, глубина, rate-limit по operation name |
| **gRPC WAF** | Анализ protobuf-сообщений, методов, metadata (экспериментально — в Envoy, Solo.io) |
| **AI/ML-модели** |  
  - Бинарная классификация (легитимный/вредоносный) на основе embedding’ов запросов  
  - NLP для анализа payload (BERT-based детекторы SQLi/XSS)  
  - Anomaly detection в time-series (частота, паттерны)  
  *(осторожно: explainability, adversarial attacks)* |

---

## 5. **Требования к хорошему WAF (checklist)**

✅ Поддержка **HTTP/2 и HTTP/3 (QUIC)**  
✅ TLS 1.2/1.3 termination и inspection (MITM для HTTPS)  
✅ Многопоточность / high-throughput (100K+ RPS)  
✅ Low latency (< 1–3 мс добавка)  
✅ Возможность кастомных правил на Lua/JS/Regexp  
✅ Тонкая настройка по URI, методу, заголовку, параметру, телу  
✅ Поддержка multipart/form-data, JSON, XML, GraphQL  
✅ Автоматическое обновление сигнатур  
✅ Dashboard и reporting (top attackers, blocked rules, FP/TP metrics)  
✅ Поддержка CI/CD: автоматическое тестирование правил (например, через `ftw` — Framework for Testing WAFs)  
✅ Соответствие стандартам: **OWASP Top 10**, **PCI DSS 6.6**, **NIST SP 800-53**, **ISO 27001**

---

## 6. **Популярные решения (open-source & commercial)**

| Категория | Продукт | Язык/платформа |
|----------|--------|----------------|
| **Open-source** | ModSecurity + NGINX/Apache | C, Lua |
| | Coraza (ModSecurity-совместимый) | Go |
| | NAXSI (NGINX) | C |
| | Sqreen (now part of Datadog) | Python/JS/Java agent |
| **Commercial** | Wallarm | Cloud/on-prem, Go-движок, advanced analytics |
| | F5 Advanced WAF (ASM) | BIG-IP, Tcl rules |
| | Cloudflare WAF | Lua (Cloudflare Workers), managed rules |
| | AWS WAF + Shield | Managed service, rule groups, regex, rate-based |
| | Akamai App Protect | На базе Signal Sciences |
| | Imperva (now Thales) | Full-stack protection |
| | Positive Technologies AppWall | Особенно популярен в РФ |

---

# Архитектурный и инфраструктуный взгляд на WAF

### **Метод: Reverse Proxy WAF (Nginx + ModSecurity / аналог)**  
*WAF как отдельный процесс/сервис на том же хосте (или кластере), через который **весь HTTP(S)-трафик проходит перед бэкендом**.*

---

### 🔧 **Общая схема**
```
Клиент → [WAF (Nginx + ModSecurity)] → [Backend: FastAPI / Django / Go / Java]
          ↑
      TLS termination здесь
      Правила: OWASP CRS + custom
      Логи: audit.log → SIEM
```

> ✅ Это — **самый распространённый, проверенный, совместимый** способ.  
> ❌ Не самый быстрый, но **достаточно производительный** (10K–50K RPS на среднем сервере).

---

## **Инфраструктурные компоненты**

| Компонент | Роль | Технологии |
|----------|------|------------|
| **Reverse Proxy** | Принимает входящие соединения, терминирует TLS, проксирует валидные запросы | Nginx, Apache HTTPD, Traefik, HAProxy |
| **WAF Engine** | Анализирует HTTP-трафик по правилам | ModSecurity (v3 — standalone), Coraza (Go), libinjection |
| **Rule Set** | Набор сигнатур и политик | OWASP CRS v3.3 / v4.5, коммерческие правила (F5, Wallarm), кастомные |
| **TLS Certificates** | Для HTTPS termination | Let’s Encrypt (`certbot`), ACM, внутренние CA |
| **Logging & Monitoring** | Фиксация решений, алертинг | `audit.log`, rsyslog → Kafka → ClickHouse / ELK; Prometheus + Grafana |

---

## **Как это работает «под капотом»**

### 1. **Приём соединения**
- Сервер слушает `0.0.0.0:443` (HTTPS).
- При установке TLS:  
  ```nginx
  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
  ```
- После рукопожатия — WAF видит **расшифрованный HTTP**.

### 2. **Обработка запроса (ModSecurity phases)**
| Фаза | Когда вызывается | Что проверяется |
|------|------------------|-----------------|
| **Phase 1** | После получения заголовков запроса | `REQUEST_URI`, `REQUEST_METHOD`, `Host`, `User-Agent` |
| **Phase 2** | После чтения тела (`Content-Length` прочитан) | `ARGS`, `POST_DATA`, `FILES`, `JSON/XML` (если `Content-Type` поддерживается) |
| **Phase 3** | До отправки заголовков ответа | `RESPONSE_STATUS`, `Location`, `Set-Cookie` |
| **Phase 4** | После тела ответа | DLP: поиск PAN, email в `RESPONSE_BODY` |
| **Phase 5** | После всего | Логгирование, метрики |

### 3. **Решение**
- Если правило сработало с `deny`, `drop`, `block` → Nginx возвращает:
  ```http
  HTTP/1.1 403 Forbidden
  Content-Type: application/json

  {"error": "Request blocked by WAF", "id": "942100"}
  ```
- Если `pass` → `proxy_pass http://127.0.0.1:8000;`.

---

## 🛠️ **Типовая конфигурация (Nginx + ModSecurity 3)**

### `nginx.conf`
```nginx
load_module modules/ngx_http_modsecurity_module.so;

http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    server {
        listen 443 ssl http2;
        server_name example.com;

        ssl_certificate     /etc/ssl/certs/example.com.pem;
        ssl_certificate_key /etc/ssl/private/example.com.key;

        location / {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

### `/etc/nginx/modsec/main.conf`
```apache
# ——— База ———
Include "/etc/modsecurity/modsecurity.conf"
SecRuleEngine On

# ——— CRS ———
Include "/etc/modsecurity/crs/crs-setup.conf"
Include "/etc/modsecurity/crs/rules/*.conf"

# ——— Кастомное правило: блокировать SQLi в параметре ?q=
SecRule ARGS:q "@rx (?i)union\s+select" \
    "id:1001,phase:2,deny,status:403,msg:'SQLi in search',logdata:'%{MATCHED_VAR}'"
```

---

## **Преимущества**

| Плюс | Обоснование |
|------|-------------|
| **Изоляция** | WAF — отдельный процесс. Падение WAF ≠ падение backend’а. |
| **Zero-code change** | Backend не знает о WAF. Можно внедрить без правок в Python/Go. |
| **Гибкость** | Можно ставить перед балансировщиком, кэшем, API gateway’ем. |
| **Полный контроль** | Ты управляешь софтом, сертификатами, правилами. |
| **Совместимость** | Работает с любым backend’ом (FastAPI, WordPress, Java, etc.). |
| **PCI DSS / ISO 27001** | Прямое соответствие требованиям (разделение слоёв). |

---

## **Недостатки и риски**

| Минус | Как смягчить |
|------|--------------|
| **Latency +1–5 мс** | Использовать `proxy_buffering off;`, `keepalive` к backend’у, `sendfile on;` |
| **SPOF (single point of failure)** | Запускать кластер: 2+ WAF-ноды + `keepalived`/`haproxy`/DNS round-robin |
| **Ложные срабатывания (FP)** | Начинать в `DetectionOnly` режиме, использовать **anomaly scoring**, tuning под твой API |
| **TLS key management** | Автоматизация через `certbot renew --post-hook "nginx -s reload"` |
| **Высокая нагрузка на CPU при парсинге multipart/JSON** | Ограничить `client_max_body_size 10m;`, отключить `SecRequestBodyAccess Off` для `/static/*` |

---

## 📈 **Производительность (оценки на 4-core, 8 GB RAM, SSD)**

| Настройка | RPS (простой GET) | RPS (POST + multipart) | Latency (p95) |
|-----------|-------------------|------------------------|---------------|
| Nginx без WAF | 45K | 12K | 3 ms |
| Nginx + ModSecurity (CRS, phase 1+2) | 18K | 4K | 8 ms |
| Nginx + ModSecurity + anomaly scoring | 12K | 2.5K | 15 ms |

> Оптимизация:  
> - `SecRequestBodyAccess On` → только для `/api/*`  
> - `SecRuleUpdateTargetById 942100 !ARGS:search` — отключить SQLi-check для `search`  
> - `SecCollectionTimeout 600` — уменьшить overhead для сессий

---

## **Безопасность: что важно настроить**

1. **Запретить прямой доступ к backend’у**:  
   ```nginx
   # На backend-сервере:
   server {
       listen 8000;
       allow 127.0.0.1;    # только с localhost
       deny all;
   }
   ```

2. **Добавить заголовки безопасности**:
   ```nginx
   add_header X-Content-Type-Options "nosniff";
   add_header X-Frame-Options "DENY";
   add_header Content-Security-Policy "default-src 'self'";
   ```

3. **Логгирование только нужного**:
   ```apache
   SecAuditLogParts ABIJDEFHZ
   SecAuditLogRelevantStatus "^(?:500|403|404|405)$"
   ```

---

## **Готовые дистрибутивы / образы**

| Вариант | Ссылка | Комментарий |
|--------|--------|-------------|
| **OWASP ModSecurity CRS Docker** | `docker run -p 80:80 -p 443:443 owasp/modsecurity-crs` | Быстрый старт |
| **Nginx + ModSecurity (Alpine)** | [github.com/SpiderLabs/ModSecurity-nginx](https://github.com/SpiderLabs/ModSecurity-nginx) | Для сборки |
| **Coraza + Traefik** | [coraza.io](https://coraza.io) | Современный Go-движок, совместимый с ModSecurity |
| **Wallarm Community** | [hub.docker.com/r/wallarm/node](https://hub.docker.com/r/wallarm/node) | Готовый WAF с дашбордом (free tier) |


---

### Метод: WAF на основе eBPF/XDP + AF_XDP

 Архитектура — по шагам (с техническими деталями)
f
```text
1. NIC (eth0)
     │
     ▼
2. [XDP program] → базовая L3/L4 фильтрация
     │   • IP blacklist, port scan, junk packets
     │   • TCP stateless: check SYN, RST, flags
     │   • UDP flood: rate per src IP
     │   • Если "maybe HTTP/HTTPS" → XDP_REDIRECT → AF_XDP UMEM
     ▼
3. [Userspace WAF app] (на C++/Rust, с AF_XDP)
     ├── zero-copy receive (xsk_ring_cons__peek + memcpy из shared UMEM)
     ├── TCP reassembly (на основе 5-tuple: src/dst IP+port + proto)
     ├── TLS 1.3 handshake parse:
     │      • Извлекаем ClientHello → SNI, JA3, ALPN, cipher suites
     │      • Решение: block (если `sqlmap`, `nmap`, `unknown SNI`), или → decrypt
     │
     ├── 🔑 **TLS MITM (расшифровка)**:
     │      • Да, в userspace **можно расшифровать**, **если у нас есть:**
     │          - Приватный ключ сервера (`server.key`)
     │          - Доступ к `Client Random` + `Server Random` + `Premaster Secret`
     │      • Как получить `Premaster Secret`?
     │          ▪ Вариант A: **RSA key exchange** (устаревший, но простой) →  
     │                перехват `ClientKeyExchange` (зашифрованный `pre_master_secret`) →  
     │                расшифровываем `RSA(privkey, ciphertext)` → получаем ключ →  
     │                генерируем `master_secret` → `key_block` → `client_write_key`, `server_write_key`.
     │          ▪ Вариант B: **(EC)DHE key exchange** (современный) →  
     │                нужно участвовать в handshake как MITM («forward proxy»):  
     │                — терминируем TLS от клиента (с нашим cert),  
     │                — устанавливаем **новое TLS-соединение** к backend’у,  
     │                — шифруем/расшифровываем трафик «на лету».  
     │                → Это **не passive sniffing**, а active proxying.  
     │                → Именно так работают Cloudflare, ModSecurity в proxy-режиме.
     │
     │      ⚠️ **Важно**: в «чистом» XDP → userspace без изменения трафика **нельзя расшифровать современный (EC)DHE TLS 1.3** без MITM.  
     │            Passive TLS decryption **невозможен** при PFS (Perfect Forward Secrecy).
     │
     ├── После расшифровки → чистый HTTP → полный WAF-анализ:
     │      • CRS-правила (SQLi, XSS, RCE)  
     │      • multipart/form-data parsing (аватары, файлы)  
     │      • DLP (поиск PAN, ИНН в теле)  
     │      • Rate limiting (на основе IP + SNI + path)
     │
     ├── Решение:
     │      • ✅ ALLOW:  
     │            — шифруем ответ (если был MITM),  
     │            — отправляем в **loopback AF_XDP ring** → `lo` интерфейс  
     │            — или через `send()` в обычный `SOCK_STREAM` сокет  
     │            — или через `XDP_TX` на виртуальный интерфейс (veth/tap)
     │      • ❌ BLOCK:  
     │            — `xsk_ring_prod__submit()` без отправки → drop  
     │            — лог в `perf_buffer` / `ringbuf` → userspace logger
```

---

### 1. **`XDP_REDIRECT` → AF_XDP — это реально и эффективно**
- Да, `XDP_REDIRECT` в `XSK_MAP` (AF_XDP socket map) — стандартный путь для zero-copy.
- Требует:
  - Ядро ≥ 5.4 (лучше ≥ 5.8),
  - NIC с поддержкой `XDP_REDIRECT` (Intel X710/X550, Mellanox ConnectX-5+, `veth`),
  - `libbpf` ≥ 1.0 или `libxdp`.

---

### 2. **Расшифровка HTTPS в userspace — возможно, но с оговорками**

| Сценарий | Возможность | Как | Требования |
|---------|-------------|-----|------------|
| **RSA key exchange (TLS ≤1.2)** | ✅ Да, passive | Перехват `ClientKeyExchange` → `RSA_decrypt(privkey)` | Устаревший, редко используется |
| **(EC)DHE (TLS 1.2+1.3)** | ❌ Passive — **нельзя**<br>✅ Active MITM — **можно** | Терминируем TLS от клиента (с нашим сертификатом), создаём новое TLS-соединение к backend’у | • Нужен доверенный сертификат (wildcard или per-SNI)<br>• Backend должен принимать от нас<br>• Возможны проблемы с pinning (HPKP, `Expect-CT`) |
| **TLS 1.3 Early Data (0-RTT)** | ⚠️ Опасно | Можно перехватить `early_data`, но он **не authenticated** → replay-атаки | Не рекомендуется разрешать 0-RTT для sensitive endpoints |

> **Вывод**:  
> Для **production-grade WAF** с HTTPS-анализом — **обязателен MITM-режим** (как у Cloudflare/ModSecurity).  
> Ты **не можешь просто «прослушать и расшифровать» современный TLS** без участия в handshake.

Но — ты **можешь отложить MITM**:
- Сначала — **L3/L4 + TLS fingerprinting (JA3)** в XDP/AF_XDP → блок 90% ботов,
- Только подозрительные → отправлять в **отдельный MITM-воркер** (на другом ядре, в отдельном процессе),
- Остальные — `XDP_PASS` → в ядро → обычный HTTPS-бэкенд (без WAF, но с baseline-защитой).

---

### 3. **Как «вернуть» пакет в сетевой стек ядра?**

Есть **три реальных способа** — и все они работают:

| Метод | Как | Плюсы | Минусы |
|------|-----|-------|--------|
| **`XDP_TX` на `veth`/`tap`** | Создаёшь `veth0` ↔ `veth1`, `veth0` в `XDP_DRV` mode, `veth1` добавляешь в `netns` backend’а. WAF → `XDP_TX` на `veth0` → пакет появляется на `veth1` → ядро → сокет. | ✅ Zero-copy возврат<br>✅ Изоляция (netns) | ❌ Требует настройки veth/tap<br>❌ +1 hop в ядре |
| **`send()` через обычный сокет** | WAF → `socket(AF_INET, SOCK_STREAM, 0)` → `connect(127.0.0.1:8000)` → `send()` | ✅ Просто, совместимо со всем | ❌ Копия в kernel (`copy_from_user`)<br>❌ +2 syscall’a на запрос |
| **`AF_XDP` → loopback ring → backend на AF_XDP** | Backend тоже использует `AF_XDP` (например, `io_uring` + `AF_XDP`). WAF → `xsk_ring_prod__fill()` → backend читает. | ✅ Full zero-copy, latency < 50 μs | ❌ Backend должен быть rewritten под AF_XDP (не все фреймворки) |

> Для **гибкости и совместимости** (FastAPI/uvicorn) — **лучше `send()` в loopback**
> Для **максимальной производительности** — **`veth + XDP_TX`** (как делает Cilium)

---

### 4. **Производительность — оценки**

| Этап | Latency | PPS на ядро (Intel X710) |
|------|---------|--------------------------|
| XDP (L4 drop) | ~0.5–2 μs | > 10 Mpps |
| AF_XDP (HTTP parse + WAF) | ~20–100 μs | 200K–1M RPS (в зависимости от правил) |
| TLS MITM (RSA) | ~100–300 μs | 10–50K RPS |
| TLS MITM ((EC)DHE, с handshake) | ~300–800 μs | 5–20K RPS |

> На слабой машине (4 ядра, без AVX) — реалистично: **50K RPS с полной WAF-проверкой + TLS MITM**.

---

## ✅ Итог: твоя архитектура — **передовая, реализуема, и ты на правильном пути**

Ты предложил **гибрид**:
- **XDP** — для ultra-fast L3/L4 pre-filter,
- **AF_XDP userspace** — для deep L7 inspection + MITM,
- **Возврат в стек ядра** — для совместимости с любым backend’ом.



---

## **Метод: Embedded / In-App WAF** 

*WAF как библиотека или агент, внедрённый **непосредственно в процесс веб-приложения**. Анализ происходит **внутри того же runtime**, что и обработка запроса.*

> 🔹 Это — **не прокси**, не отдельный сервис, а **часть приложения**.  
> 🔹 Работает без сетевых задержек, но требует интеграции в код.  
> 🔹 Идеально для микросервисов, serverless, edge-functions, high-security сред.

---

## **Общая схема**

```
Клиент → [Reverse Proxy / Load Balancer] → [Backend Process]  
                                                ↑  
                                           ┌────┴────┐  
                                           │  WAF    │ ← lib / agent  
                                           │  (in-process)  
                                           └─────────┘  
                                           │  RASP, context-aware  
                                           ▼  
                                      [Business Logic]
```

> ✅ Трафик **не покидает процесс** → latency ≈ 0.  
> ❌ Ошибка в WAF = падение всего приложения (segfault, OOM).

---

## **Типы реализации (по языку и подходу)**

| Тип | Как работает | Примеры |
|-----|--------------|---------|
| **Native library (C/C++)** | Подключается через FFI/`dlopen`/статическая линковка. Вызывается на каждом запросе. | `libmodsecurity3` (standalone), `libinjection`, `hyperscan` |
| **Language-native agent** | Инжектится в runtime через instrumentation (monkey-patching, bytecode injection). | `Datadog AppSec` (Python/JS/Java), `Sqreen` (deprecated), `Contrast Security` |
| **Middleware / Decorator** | Явная вставка WAF-логики в middleware-цепочку фреймворка. | FastAPI `@app.middleware`, Django `MIDDLEWARE`, Express `app.use(waf)` |
| **WebAssembly (WASM)** | Правила компилируются в `.wasm`, выполняются в sandbox’е внутри приложения. | `proxy-wasm` (Envoy), `wasmtime` + `Coraza`, `Fastly Compute@Edge` |

---

## **Преимущества**

| Плюс | Обоснование |
|------|-------------|
| **Минимальная latency** | Нет network hop’ов, копирования памяти, контекст-свичей. |
| **Deep context awareness** | Можно проверять: *залогинен ли пользователь?*, *роль админ?*, *ID сессии валидный?* → RASP-логика. |
| **Легко в CI/CD** | WAF — часть приложения → тестируется как обычный код (`pytest`, `coverage`). |
| **Идеален для serverless / edge** | Vercel, Cloudflare Workers, AWS Lambda — нет места для прокси. |
| **Гибкость** | Можно писать правила на Python:  
  ```python
  if "admin" in request.url.path and not request.user.is_admin:
      block("Unauthorized admin access")
  ```

---

## **Недостатки и риски**

| Минус | Как смягчить |
|------|--------------|
| **WAF = часть приложения** | Segfault в `libmodsecurity` → падение всего worker’а. → Использовать **separate process pool** или `multiprocessing`. |
| **Языковая привязка** | Нет единого решения для Python/Go/Java. → Выбирать **WASM-based** (например, Coraza + `wasmtime`). |
| **Трудно масштабировать правила** | Обновление CRS требует деплоя приложения. → Выносить правила в `ConfigMap`/Consul, hot-reload через `SIGHUP`. |
| **Нет TLS termination** | HTTPS уже расшифрован — но ты **не контролируешь сертификаты**. → Ставить WAF **после** reverse proxy (Nginx), но **до** бизнес-логики. |
| **Сложнее логгирование** | Логи WAF смешиваются с логами приложения. → Отправлять в отдельный `structlog` logger с `logger.bind(waf=True)`. |

---

## **Производительность (оценки на 4-core, Python 3.11, uvicorn)**

| Сценарий | RPS | Latency (p95) | CPU / запрос |
|----------|-----|---------------|--------------|
| Без WAF | 12K | 4 ms | 0.08 ms |
| Middleware (libmodsecurity3, phase 1+2) | 6.5K | 7 ms | 0.35 ms |
| Middleware (hyperscan + простые regex) | 9K | 5 ms | 0.15 ms |
| RASP (проверка ORM-запросов) | 4K | 12 ms | 0.8 ms |

> Оптимизация:  
> - Использовать `@lru_cache` для часто встречающихся паттернов,  
> - Отключать `process_request_body` для `GET /static/*`,  
> - Парсить JSON только при `Content-Type: application/json`.

---

## **Безопасность: особенности embedded-WAF**

### 1. **RASP-возможности (уникальные для in-app)**
| Возможность | Пример |
|-------------|--------|
| **SQL Injection в ORM** | Перехват `cursor.execute(sql, params)` → проверка `sql` на `UNION SELECT` |
| **Path Traversal в open()** | Monkey-patch `open()` → проверка пути на `../` |
| **SSRF в requests.get()** | Wrapper над `requests` → блок `127.0.0.1`, `metadata.google.internal` |
| **Unsafe deserialization** | Hook `pickle.loads(data)` → block if `__reduce__` содержит `os.system` |

### 2. **Защита от bypass’а**
- **Нельзя просто отключить middleware** → инжектить через `LD_PRELOAD` или `sitecustomize.py`.
- **Контроль целостности**: хэш бинарника WAF в startup-check:
  ```python
  assert hashlib.sha256(open("waf.so", "rb").read()).hexdigest() == EXPECTED_HASH
  ```

---

## **Будущее embedded-WAF**

- **WASM-правила**: правила — `.wasm`, обновляются без пересборки приложения.
- **eBPF + userspace agent**: XDP фильтрует 90% junk, а embedded WAF — делает deep inspection в приложении.
- **AI в runtime**: модель на TensorFlow Lite Micro — inference < 100 μs для anomaly detection.

---


## **Метод: Cloud / SaaS WAF (через DNS и reverse proxy в облаке)**

*WAF как **управляемый сервис**, размещённый **вне вашей инфраструктуры**, в точках присутствия (PoP) провайдера. Интеграция — через изменение DNS-записей.*

> 🔹 Это — **не софт, который ты ставишь**, а **сервис, который ты включаешь**.  
> 🔹 Ты **не управляешь серверами**, только правилами и политиками.  
> 🔹 Лидеры: **Cloudflare**, **AWS WAF + CloudFront**, **Akamai App Protect**, **Fastly Next-Gen WAF**, **Imperva (Thales)**.

---

## **Как это работает? Почему DNS?**

### Главный принцип: **DNS как «переключатель трафика»**

Ты **не меняешь код**, **не ставишь софт**, **не трогаешь сервера**.  
Ты просто **меняешь DNS-запись**, и весь трафик **автоматически проходит через облако провайдера**.

#### Пример:
Было:
```dns
example.com.  IN  A  203.0.113.10   ; твой сервер в ДЦ
```

Стало:
```dns
example.com.  IN  A  104.21.5.78    ; Cloudflare PoP (Санкт-Петербург)
example.com.  IN  A  172.67.169.45  ; Cloudflare PoP (Москва)
```

→ Теперь **все запросы к `example.com` идут не к тебе напрямую**, а в **ближайший дата-центр Cloudflare** → там — WAF, DDoS-защита, CDN, Bot Fight Mode → и **только потом** — к твоему серверу (`203.0.113.10`), **по защищённому туннелю**.

---

## **Архитектура под капотом (на примере Cloudflare)**

```text
1. Пользователь (Москва)
     │
     ▼
2. DNS-запрос → Cloudflare DNS (1.1.1.1) → отдаёт IP ближайшего PoP
     │
     ▼
3. TCP handshake → TLS 1.3 → [Cloudflare PoP: SVO / DME / LED]
     │   • TLS termination (сертификат от Cloudflare или загруженный тобой)
     │   • Проверка IP по threat intel (Spamhaus, AbuseIPDB, внутренние)
     │   • Rate limiting (счётчики в распределённом Redis)
     │   • WAF движок (на Lua, внутри Nginx-подобного сервера)
     │   • Если подозрение → JS-челлендж (Turnstile), CAPTCHA, Managed Challenge
     │
     ▼
4. Решение:
     ├── ✅ ALLOW → HTTPS-туннель (Argo Smart Routing) → твой origin (203.0.113.10:443)
     └── ❌ BLOCK → 403 / JS challenge / silent drop
```

> **Важно**: твой сервер (`origin`) **никогда не видит IP атакующего** — только IP Cloudflare.  
> Чтобы получить реальный IP, бэкенд должен читать заголовок:  
> ```http
> CF-Connecting-IP: 95.167.123.45
> ```

---

## **Ключевые компоненты Cloud WAF**

| Компонент | Как работает | Реализация (Cloudflare) |
|----------|--------------|------------------------|
| **PoP (Point of Presence)** | Физический дата-центр (их >300 по миру). Трафик идёт в ближайший. | Anycast BGP: `AS13335` объявляет `104.16.0.0/12` в 300+ точках |
| **TLS Termination** | Расшифровка HTTPS у провайдера. Ты загружаешь сертификат или используешь Universal SSL. | `ECDSA P-256`, `RSA 2048+`, поддержка TLS 1.3 (0-RTT отключён по умолчанию) |
| **WAF Engine** | HTTP-анализ по правилам. Скорость — критична (миллионы RPS). | Высокопроизводительный Nginx-fork + LuaJIT + JIT-компиляция правил |
| **Threat Intelligence** | Базы вредоносных IP, UA, ASN, TOR-exit, сканеров. Обновление — каждые 5 мин. | Внутренние (CF Radar) + внешние (Spamhaus, AbuseIPDB, Cisco Talos) |
| **Bot Management** | Обнаружение headless-браузеров, Selenium, Puppeteer. | TLS/JA3 fingerprinting, Canvas/WebGL fingerprint, поведенческий анализ |
| **Origin Connect** | Защищённое соединение до твоего сервера. | Argo Tunnel (cloudflared) или IP ACL + Mutual TLS |

---

## **Как настраивается? (UI/API/CLI)**

### 1. **DNS Setup (обязательно)**
- В панели Cloudflare:  
  `example.com` → **DNS** → `A` запись → статус **Proxied (orange cloud)**  
  → трафик идёт через Cloudflare.

- Grey cloud = DNS only (без WAF/CDN/DDoS).

### 2. **SSL/TLS Mode**
| Режим | Описание | Когда использовать |
|------|----------|-------------------|
| **Off** | HTTP only | Никогда (небезопасно) |
| **Flexible** | HTTPS ←→ CF, CF → HTTP origin | Legacy backend без HTTPS |
| **Full** | HTTPS ←→ CF, CF → HTTPS origin (без проверки cert) | Твой backend на HTTPS, но самоподписанный cert |
| **Full (strict)** | HTTPS ←→ CF, CF → HTTPS origin + проверка cert | Production, доверенный CA (Let’s Encrypt) |

### 3. **WAF Rules (Managed + Custom)**
#### Managed Rulesets (готовые)
| Набор | Покрытие | Пример правила |
|-------|----------|----------------|
| **Cloudflare Managed Ruleset** | OWASP Top 10 + zero-day | `100000: SQLi in WordPress wp-login.php` |
| **OWASP ModSecurity Core Rule Set (CRS)** | Полный CRS v3.3 | `942100: SQL Injection Attack Detected` |
| **Cloudflare Specials** | Уязвимости в популярных CMS | `200001: WordPress XML-RPC brute force` |

#### Custom Rules (пишешь сам)
```lua
-- Пример: блокировать запросы с SQLi-паттерном в ?q=
(ip.geoip.country eq "RU" and http.request.uri.query contains "q=" and 
 http.request.uri.query matches "(?i)union\\s+select") 
→ Block
```

Можно настраивать по:
- `ip`, `ssl`, `http.request`, `cf.threat_score`, `cf.bot_management.score`

### 4. **Rate Limiting**
```yaml
# Пример: 100 запросов/мин на /api/login, с burst=20
path: "/api/login"
period: 60
requests_per_period: 100
mitigation: "challenge"  # или block, js_challenge
```

### 5. **Bot Fight Mode / Super Bot Fight Mode**
- Анализирует:  
  - `User-Agent` (sqlmap, nmap, python-requests)  
  - `Accept` / `Accept-Language` несоответствия  
  - TLS handshake (JA3 hash)  
  - Время загрузки JS-челленджа  
- Может:  
  - `log` (только лог),  
  - `challenge` (JS проверка),  
  - `block` (403).

---

## **Преимущества**

| Плюс | Обоснование |
|------|-------------|
| **Мгновенное включение** | DNS change → через 60 сек защита включена. Без downtime. |
| **Глобальная DDoS-защита** | 200+ Tbps capacity (Cloudflare), L3-L7 атаки — «гасятся» в PoP. |
| **Zero maintenance** | Ты не обновляешь ядро, не патчишь OpenSSL, не масштабируешь WAF-ноды. |
| **Threat intel «из коробки»** | Блокировка IP из 100+ источников, обновление каждые 5 минут. |
| **Edge-compute** | Workers (JS/WASM) → кастомная логика прямо на PoP:  

---

## **Недостатки и риски**

| Минус | Как смягчить |
|------|--------------|
| **Зависимость от провайдера** | Падение Cloudflare = downtime. → Иметь **fallback DNS** (grey cloud) в Route53 Failover. |
| **Latency до origin** | Пользователь → CF PoP → твой сервер (может быть +20–100 мс). → Использовать **Argo Smart Routing** (оптимизированные пути). |
| **Ограниченная кастомизация** | Нельзя писать правила на C/Rust. → Использовать **Workers** для сложной логики. |
| **Цена при высоком трафике** | $200+/мес за Pro, $2000+ за Enterprise. → Для малых проектов — бесплатный тариф (ограниченные правила). |
| **Юридические риски** | Трафик проходит через США/ЕС → GDPR, ФЗ-152. → Включить **Regional Services** (Cloudflare EU-only). |
| **Нельзя полный MITM control** | Сертификаты управляются CF → нельзя вставить свой CA в trust store. → Использовать **Mutual TLS** до origin. |

---

## **Производительность и лимиты (Cloudflare Free/Pro)**

| Метрика | Free | Pro ($20/мес) | Enterprise |
|--------|------|---------------|------------|
| **Правила WAF** | 5 custom | 20 custom | 1000+ |
| **Rate limiting rules** | 1 | 10 | 1000+ |
| **Managed rulesets** | Только CF Managed | + OWASP CRS | + Custom Managed |
| **Bot Fight Mode** | ❌ | ✅ | ✅ + Super |
| **Workers** | 100K req/day | 10M req/day | ∞ |
| **SSL/TLS** | Universal SSL (3 мес) | Custom certs, TLS 1.3 | Mutual TLS, Client Certs |
| **Origin timeout** | 100 сек | 600 сек | Custom |

> **Совет**: На бесплатном тарифе можно заблокировать 90% атак через:  
> - Managed Ruleset (включить все «High» и «Medium» правила),  
> - Firewall Rules: `cf.threat_score > 15 → challenge`,  
> - Security Level: **High** (строже проверки).

---

## **Безопасность: что важно настроить**

### 1. **Origin Protection**
- Закрой прямой доступ к серверу:
  ```nginx
  # На твоём origin’е:
  allow 173.245.48.0/20;   # Cloudflare IPs (ipv4)
  allow 103.21.244.0/22;
  deny all;
  ```
  → Список всех IP: [https://www.cloudflare.com/ips/](https://www.cloudflare.com/ips/)

### 2. **Заголовки безопасности**
Cloudflare автоматически добавляет:
```http
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Referrer-Policy: strict-origin-when-cross-origin
```
→ Можно дописать в **Transform Rules**:
```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'
```

### 3. **Логгирование**
- **Security Events** — все блокировки (в UI / API / Logspush to Kafka/S3).
- **Audit Log** — кто и что менял в настройках.
- **Logpull API** — выгрузка в SIEM:
  ```bash
  curl -H "X-Auth-Email: you@edgecenter.ru" \
       -H "X-Auth-Key: xxx" \
       "https://api.cloudflare.com/client/v4/zones/ZONE_ID/logs/received?start=..."
  ```


---

## **Сравнение провайдеров**

| Провайдер | Плюсы | Минусы | Для кого |
|----------|-------|--------|----------|
| **Cloudflare** | Бесплатный тариф, Workers, DDoS 200+ Tbps | Ограниченные правила на Free | Стартапы, блоги, SaaS |
| **AWS WAF + CloudFront** | Интеграция с ALB, Lambda@Edge, IAM | Дорого при высоком RPS ($5/мес за rule) | AWS-стэки, enterprise |
| **Akamai App Protect** | RASP-like, behavioral analytics | Очень дорого ($10K+/мес) | Банки, правительство |
| **Fastly Next-Gen WAF** | WASM-правила, VCL, real-time logs | Нет бесплатного тарифа | High-load API, media |
| **Imperva (Thales)** | Full-stack (WAF + RASP + DDoS) | Сложный onboarding | Legacy enterprise |

---

## **Когда выбирать Cloud WAF?**

**Выбирай, если**:
- Ты хочешь **защиту «за 5 минут»** без инженерных усилий,
- У тебя **глобальная аудитория** (нужен CDN + низкая latency),
- Ты подвержен **DDoS-атакам** (L3-L7),
- Нет команды для сопровождения on-prem WAF,
- Готов платить за managed-сервис.

**Не выбирай, если**:
- Ты в **госсекторе/финансах** с запретом на обработку трафика за рубежом,
- Требуется **полный контроль над TLS MITM** (например, для внутренних CA),
- У тебя **очень специфичные правила**, которые нельзя выразить в UI/API,
- Ты работаешь **на edge-устройствах** без выхода в интернет.

---

## **Будущее Cloud WAF**

- **WASM-правила в Workers**: загрузка `.wasm` с кастомным движком (Coraza, libinjection).
- **AI-driven WAF**: автоматическое создание правил по логам (Cloudflare Radar AI).
- **Zero Trust Integration**: WAF + SASE (Secure Access Service Edge) — проверка устройства, пользователя, контекста.
- **Confidential Computing**: MITM с защищённой памятью (Intel SGX) — ключи не видны даже провайдеру.
