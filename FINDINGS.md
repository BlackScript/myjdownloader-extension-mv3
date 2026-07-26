# Wissensbasis: MV3-Portierung, Captcha-Pfade, offene Baustellen

Stand: 26.07.2026 · Branch `notes/findings` (nur im Fork `BlackScript/…`, **nie** Teil eines Upstream-PRs).

Diese Datei hält fest, was wir bei der Analyse von Issue #5 (`magnetgrouplabs`) verifiziert haben — inklusive
der Quellen, damit nichts erneut geraten werden muss. Getrennt in **belegte Fakten** und **offene Punkte**.
Offene Punkte haben je ein Issue im Fork.

---

## 1. Belegte Fakten — MyJDownloader-Webinterface (`my.jdownloader.org`)

### 1.1 Extension-Erkennung per Einmal-Handshake
Die Seite registriert einen `message`-Listener und postet **genau einmal** einen Ping:

```js
window.addEventListener('message', function (e) {
  if (e.data !== undefined && e.data.type === 'ping' && e.data.name === 'pong') {
    window.jd.extensionInstalled = true;
  }
}, false);
window.postMessage({ name: 'ping' }, '*');
```

Fundstelle: GWT-Permutation `myjdownloader/<PERMUTATION>.cache.js` (Funktionen `TTd` = Listener, `uTd` = Ping,
Konstanten `UKi='ping'`, `Tmi='message'`, `rBi='*'`). Permutationen stehen in `myjdownloader/myjdownloader.nocache.js`.

Unsere Antwort kommt aus `contentscripts/webinterfaceEnhancer.js` (`{type:'ping', name:'pong', data:{version}}`).

### 1.2 Wer `jd.extensionInstalled` ausliest — und was daran hängt
Das Flag ist **nicht** kosmetisch. Gelesen wird es im ausgelagerten GWT-Fragment
`myjdownloader/deferredjs/<PERMUTATION>/2.cache.js`: `function RTd() { return $wnd.jd.extensionInstalled; }`.

Aufrufer im Captcha-Presenter verzweigen für `RecaptchaV2Challenge` (`SRi='RecaptchaV2Challenge'`, `TRi='rawtoken'`):

| Stelle | Bedeutung |
|--------|-----------|
| `HMe(...)` | `captcha/get`-Request bekommt `rawtoken` **nur** mit gemeldeter Extension |
| `VNe(...)` | Extension-Pfad zum Laden des Captchas nur mit Flag, sonst anderer Zweig |
| `kSe(...)` | `!RTd()` setzt einen Nicht-Extension-Modus in der Challenge-Ansicht |
| DONE-Zweig | ein Abschluss-Schritt läuft nur mit Flag |

`rawtoken` ist derselbe Parameter, den unser `scripts/services/Rc2Service.js:308` beim `/captcha/get` anfordert.
**Ohne Pong gibt das Webinterface reCAPTCHA-v2-Jobs also gar nicht an die Extension.**

**Falle:** `tm/enhance.js` der Seite setzt `window.jd.extensionInstalled = true` selbst — bricht aber in Zeile 8 mit
`if (!isDebug) return;` ab (`debug=true`-Cookie). Für normale Nutzer also wirkungslos. Wer nur den Setter sieht und
den Cookie-Check überliest, schließt falsch, dass das Flag immer gesetzt ist.

### 1.3 Webinterface-Link im Popup ist korrekt aufgebaut
`https://my.jdownloader.org/?deviceId=<encodeURIComponent(device.id)>#webinterface:downloads`
(`partials/templateCache.js` `mydevice.html`, `scripts/controllers/DeviceController.js` `getEncodedDeviceId`).

Die Seite baut ihren eigenen Link identisch: `UrlBuilder.setParameter('deviceId', device.id)` + Hash
`webinterface:downloads` (GWT `IOd`, `D8c` liefert `device.id`). Beim Lesen macht sie
`getParameter('deviceId').replace('/', '')` — relevant nur für IDs mit Slash; normale IDs sind 32 Hex-Zeichen.
Die Geräteliste im Popup kommt **live** aus `listdevices` (`ConnectedController.loadDevices`), nicht aus
`CACHED_DEVICE_LIST` (der wird nur für Toolbar/Geräteauswahl benutzt).

---

## 2. Belegte Fakten — JDownloaders eigener Browser-Solver

Quelle (öffentliche SVN-Spiegel): `mirror/jdownloader` bzw. `mycodedoesnotcompile2/jdownloader_mirror`
unter `src/org/jdownloader/captcha/v2/`.

### 2.1 URL-Schema und Verben
`BrowserReference.java` bedient auf dem Loopback-Interface:
`http://127.0.0.1:<port>/captcha/<typ>/<siteDomain>?id=<challengeId>` mit `do=loaded`, `do=solve&response=`,
`do=skip&skiptype=` auf derselben URL. Zusätzlich liest es den Header `X-Myjd-Appkey`.

### 2.2 Der wichtigste Unterschied: pro Challenge-Typ
- **`recaptcha/v2/recaptcha.html`** bindet das Widget ein und schickt den Token selbst:
  `xhr.open("GET", window.location.href + "&do=solve&response=" + document.getElementById("g-recaptcha-response").value)`.
  → Die Extension ist hier **nicht** nötig; `contentscripts/captchaSolverContentscript.js` deckt es ab.
- **`hcaptcha/hcaptcha.html`** enthält **kein hCaptcha-Widget und lädt `api.js` nie** (der String `hcaptcha` kommt
  außerhalb der Base64-Bilder nicht vor). hCaptcha muss auf der Hoster-Domain laufen; die Seite ist nur die
  „Browser extension required"-Seite. → Für hCaptcha ist die Extension der **einzige** Weg.

Damit ist die Notiz in `.planning/backups/native-captcha-approach/WEB-TAB-CAPTCHA-VALIDATION.md`
(„the page is designed to be functional standalone", MEDIUM confidence) nur für reCAPTCHA richtig.

### 2.3 Job-Daten in Meta-Tags (von JD serverseitig gefüllt)
`sitekey`, `sitekeyType`, `v3action`, `challengeType`, `optionals`, `boundToDomain`, `sameOrigin`, `siteDomain`,
`sToken`, `enterprise`, `siteUrl`, `favIcon`, `challengeId` — ungefüllt sehen sie aus wie `%%%sitekey%%%`.

### 2.4 Keine Extension-Erkennung auf JDs Seiten
JDs Challenge-Seiten prüfen **nicht**, ob eine Extension da ist. Das JS dort erkennt nur den User-Agent
(`isOpera`/`isFirefox`/`isChrome`/`noExtensionBrowser`/`unknown`), um zu entscheiden, **welche** Install-Box
angezeigt wird. Der ping/pong aus 1.1 gilt also ausschließlich für `my.jdownloader.org`.

---

## 3. Status der MV3-Lücken

Ursache der meisten Lücken: In MV2 hing die Verdrahtung an der Background-Page (`scripts/controllers/BackgroundController.js`).
Unter MV3 wird dieser Angular-Controller nie instanziiert (`popup-app.js` hat bewusst keine `/`-Route auf `BackgroundCtrl`),
er war also seit der Portierung toter Code und wurde in `8484676` entfernt. Angular-Services sind **lazy** — ohne Injektor
läuft ihre Factory nie.

| # | Lücke | Status |
|---|-------|--------|
| 1 | `webinterface-enhancer/settings` hatte in MV3 keinen Responder → `active` blieb false → kein Pong | **gefixt** in Upstream-PR #18 |
| 2 | Übergabe von JDs hCaptcha-Seite fehlte komplett (MV2: `browserSolverEnhancer.js`) | **gefixt** in Upstream-PR #19 |
| 3 | `myjdCaptchaSolver.js` hardcodete `callbackUrl: 'MYJD'` an 4 Stellen | **gefixt** in Upstream-PR #19 |
| 4 | `Rc2Service` wird nie instanziiert → Webinterface-Captcha-Pfad hat keinen Auslöser | **offen** → Fork-Issue |
| 5 | `myjdrc2/captcha-new` hat in MV3 keinen Sender; `webinterface-enhancer/captcha-done` keinen Empfänger | **offen** → Teil von #4 |
| 6 | Badge „!" beim Browserstart, Warmstart-Retry via `setTimeout`, toter `myjd_connection_state`-Listener | **offen** → Fork-Issue |
| 7 | Geräteliste ungefiltert → tote Registrierungen erscheinen als Karte mit unbrauchbarem Webinterface-Link | **offen** → Fork-Issue |
| 8 | `X-Myjd-Appkey` sendet 4-teilige Version, JDs Parser akzeptiert nur 3 Teile | **offen** → Fork-Issue |
| 9 | Nach `#rc2jdt`-Navigation meldet `captchaSolverContentscript.js` zusätzlich einen Solve mit `callbackUrl: null` | **offen**, kosmetisch → Teil von #4 |

### 3.1 Was in `Rc2Service.js` steckt (für den Port nach #4)
- `:42` `webRequest`-Erkennung von JDs lokalem Solver (`http://127.0.0.1:\d+/captcha/(recaptchav(2|3)|hcaptcha)/…?id=\d+`)
- `:297` `tabs.onUpdated`-Erkennung von `#rc2jdt&c=<captchaId>` (Webinterface-Flow)
- `:302–312` Jobabruf über die Device-API: `/captcha/getCaptchaJob` → `/captcha/get` mit `rawtoken`
- `:69`/`:241` Senden von `myjd-prepare-captcha-tab` (der Handler dazu **existiert** in `background.js`)
- `:92–105` direkter Rückweg `/captcha/solve` über die Device-API
- `:173–332` `myjdrc2`-Listener (`loaded`, `tabmode-init`, `response`, `mouse-move`, `captcha-new`, `captcha-get`, `tabmode-skip-request`)
- `:50` `onLoginNeeded` → leitet den Tab auf `loginNeeded.html`

### 3.2 Test-Blindstellen (wichtig für künftige Fixes)
- `scripts/services/__tests__/Rc2Service.test.js`, `background-captcha.test.js`, `captchaSolverContentscript.test.js`
  prüfen **nur Quelltext per Regex**. Deshalb ist CI grün, obwohl der Code nie ausgeführt wird. Neue Fixes
  brauchen ausführende Tests (Muster: `contentscripts/__tests__/cnlInterceptorMain.test.js` mit `new Function(...)`).
- jsdoms `AbortSignal` hat kein statisches `timeout()`, das Chrome hat und `background.js` bei jedem `fetch` an JD
  nutzt. Ungepatcht wirft das `TypeError`, der `catch` im Handler frisst es → der komplette Rückweg wirkt lautlos
  wirkungslos. Stub liegt seit PR #19 in `jest.setup.js`. Gleiche Klasse wie XHR-im-Service-Worker aus PR #8.

---

## 4. Arbeitsweise mit dem Upstream
- PRs gehen gegen **`dev`**, ein Thema pro PR, `npx jest` muss grün sein, Live-Verifikation beschreiben
  (siehe `CONTRIBUTING.md` auf `dev`). `master` = letzter Release.
- Fine-grained `GH_TOKEN` darf in fremden Repos **nicht** schreiben (keine PRs/Issues) → Classic-Token nutzen.
- Upstream prüft Behauptungen nach (siehe Review zu PR #8: XHR existiert im Service Worker nicht) — also nur
  belegte Aussagen, Unverifiziertes explizit als solches kennzeichnen.
