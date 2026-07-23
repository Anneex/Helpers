# Прозорі відео на вебі — хелпер

Робочий гайд по підготовці alpha-відео (прозорий фон) для сайту + вирізання по формі з Figma.

---

## Чому два формати

Прозорість у відео браузери підтримують по-різному:

| Формат | Кодек | Для кого |
|--------|-------|----------|
| **WEBM** | VP9 (`libvpx-vp9`) | Chrome, Firefox, Edge |
| **MOV**  | HEVC (`hevc_videotoolbox`) | Safari (macOS/iOS) |

Універсального формату немає — треба віддавати обидва й вибирати потрібний на льоту (див. блок JS нижче).

---

## Тег `<video>`

```html
<video muted playsinline autoplay loop style="display:block; width:100%; height:100%;"></video>
```

- `muted` + `autoplay` — без mute браузер блокує автозапуск
- `playsinline` — щоб на iPhone не відкривалось у фулскрін
- `loop` — за потреби

---

## ffmpeg — базова оптимізація (відео вже прозоре)

**MOV для Safari:**
```bash
ffmpeg -y -i key.mov -c:v hevc_videotoolbox -alpha_quality 0.75 -pix_fmt bgra -tag:v hvc1 -q:v 65 -an key_hevc_alpha.mov
```

**WEBM для решти:**
```bash
ffmpeg -y -i key.mov -c:v libvpx-vp9 -pix_fmt yuva420p -an key_optimized.webm
```

**WEBM з кращою компресією (constant quality):**
```bash
ffmpeg -y -i key.webm -c:v libvpx-vp9 -pix_fmt yuva420p -auto-alt-ref 0 -crf 35 -b:v 0 -an key_optimized.webm
```

**MP4 без прозорості (звичайне непрозоре відео):**
```bash
ffmpeg -y -i key.mp4 -c:v libx264 -crf 28 -preset slow -pix_fmt yuv420p -movflags +faststart -an key_optimized.mp4
```

### Ключові флаги
- `-c:v` — відеокодек (`libvpx-vp9` = VP9, `hevc_videotoolbox` = Apple HEVC, `libx264` = H.264)
- `-pix_fmt yuva420p` / `bgra` — формат з **alpha** (буква `a`); `yuv420p` без alpha
- `-crf` — якість, менше = краще й важче (робочі 23–35); `-q:v` — те саме для апаратного енкодера
- `-tag:v hvc1` — обов'язковий тег, без нього Safari не читає HEVC
- `-alpha_quality 0.75` — якість саме alpha-каналу
- `-movflags +faststart` — відео грає до повного завантаження
- `-an` — прибрати звук (для декору не потрібен)
- `-y` — перезаписати вихідний файл без питання

---

## ffmpeg — конвертація MP4 → MOV / WEBM / WebP

> **Важливо:** якщо вхідний MP4 **непрозорий**, конвертація сама по собі прозорість не додасть — вийде той самий непрозорий контент в іншому контейнері. Щоб отримати alpha, потрібна маска (див. наступний розділ) або вихідник, у якому alpha вже є.

### MP4 → MOV (HEVC, для Safari)
```bash
# with alpha (source must already have transparency)
ffmpeg -y -i input.mp4 -c:v hevc_videotoolbox -alpha_quality 0.75 -pix_fmt bgra -tag:v hvc1 -q:v 65 -an output.mov
```
```bash
# without alpha (simple repack to MOV)
ffmpeg -y -i input.mp4 -c:v hevc_videotoolbox -pix_fmt yuv420p -tag:v hvc1 -q:v 65 -an output.mov
```
```bash
# fastest: no re-encode, just change container (keeps original quality)
ffmpeg -y -i input.mp4 -c copy output.mov
```

### MP4 → WEBM (VP9, для Chrome/Firefox)
```bash
# with alpha (source must already have transparency)
ffmpeg -y -i input.mp4 -c:v libvpx-vp9 -pix_fmt yuva420p -auto-alt-ref 0 -crf 32 -b:v 0 -an output.webm
```
```bash
# without alpha, good compression
ffmpeg -y -i input.mp4 -c:v libvpx-vp9 -pix_fmt yuv420p -crf 32 -b:v 0 -row-mt 1 -an output.webm
```

`-row-mt 1` вмикає багатопотокове кодування — VP9 інакше рахує дуже довго.

### MP4 → WebP (анімований)
Формат для коротких лупів замість GIF: менша вага, підтримує прозорість.

```bash
# animated WebP loop, scaled down to 800px wide
ffmpeg -y -i input.mp4 -vf "fps=20,scale=800:-1:flags=lanczos" -c:v libwebp -lossless 0 -q:v 70 -loop 0 -an output.webp
```

- `-loop 0` — нескінченний цикл
- `-q:v` — якість 0–100 (більше = краще, на відміну від `-crf`)
- `-lossless 1` — без втрат, якщо потрібен ідеальний край прозорості (файл важчий)
- `fps=20` + `scale` — обов'язково зменшуй, інакше WebP розпухне; підходить лише для коротких роликів (2–5 с)

> Для довгих або великих відео WebP не варіант — бери WEBM/MOV.

---

## ffmpeg — вирізати відео по формі з Figma

Figma **не експортує відео** — тільки статичний poster-кадр. Тому маску накладаємо вручну через ffmpeg АБО через CSS (див. останній розділ — для вебу зазвичай краще CSS).

**Крок 1.** У Figma вибери форму (напр. `Vector 794`), білу заливку на прозорому фоні → експортуй PNG **у розмір відео** (краще з запасом, множник 3–4x, інакше край буде мутний).

**Крок 2.** Дізнайся розмір відео:
```bash
ffmpeg -i brain.mp4      # дивись рядок "Stream ... Video: ... ШИРИНАxВИСОТА"
```

**Крок 3 — WEBM (ручний розмір):**
```bash
# replace 1950:2100 with your video's real dimensions
ffmpeg -i brain.mp4 -i mask.png -filter_complex "[1:v]scale=1950:2100,format=gray[m];[0:v]format=yuva420p[v];[v][m]alphamerge[out]" -map "[out]" -c:v libvpx-vp9 -pix_fmt yuva420p brain_masked.webm
```

**Крок 3 — MOV (ручний розмір):**
```bash
ffmpeg -i brain.mp4 -i mask.png -filter_complex "[1:v]scale=1950:2100,format=gray[m];[0:v]format=yuva420p[v];[v][m]alphamerge[out]" -map "[out]" -c:v hevc_videotoolbox -alpha_quality 0.75 -tag:v hvc1 -q:v 65 brain_masked.mov
```

**Універсальний варіант — маска підганяється під відео автоматично** (не треба вбивати цифри):
```bash
# scale2ref auto-matches the mask to the video's exact size
ffmpeg -i brain.mp4 -i mask.png -filter_complex "[1:v][0:v]scale2ref=iw:ih[m][v0];[m]format=gray[m2];[v0]format=yuva420p[v];[v][m2]alphamerge[out]" -map "[out]" -c:v libvpx-vp9 -pix_fmt yuva420p brain_masked.webm
```

> `alphamerge` бере яскравість маски (біле = видиме, чорне/прозоре = вирізане) і робить з неї alpha-канал відео.

---

## Часті помилки

| Помилка в терміналі | Причина / фікс |
|---|---|
| `Input frame sizes do not match (AxB vs CxD)` | Розмір маски ≠ розмір відео. Постав правильний `scale=W:H` або юзай `scale2ref` |
| `No such file or directory` | Невірна назва файлу. Перевір через `ls`, зайди в папку через `cd` |
| Зайві `\` в одному рядку ламають команду | Слеші — це перенос рядка. У однорядковій команді їх бути НЕ повинно |
| Safari не грає `.mov` | VP9 не працює в MOV. Для MOV потрібен `hevc_videotoolbox` + `-tag:v hvc1` |
| `No accelerated colorspace conversion...` | НЕ помилка, просто інфо. Конвертація йде на CPU, результат нормальний |
| Мутний/зубчастий край після вирізу | Маска замала. Переекспортуй PNG з Figma більшого розміру |

---

## Альтернатива для вебу — маска через CSS (рекомендовано)

Замість запікання маски у відео — тримай відео прямокутним (навіть **непрозорий MP4**), а форму наклади маскою в браузері. Край буде чіткий (векторна маска), не треба alpha-відео взагалі.

1. У Figma: форма → правий клік → **Copy as SVG** (або Export → SVG).
2. Розмітка:

```html
<div class="masked-video">
  <video src="brain.mp4" muted playsinline autoplay loop></video>
</div>
```

```css
.masked-video {
  /* SVG silhouette clips the video to shape */
  -webkit-mask: url('brain-shape.svg') center / contain no-repeat;
  mask: url('brain-shape.svg') center / contain no-repeat;
}
.masked-video video {
  display: block;
  width: 100%;
  height: 100%;
}
```

**Коли CSS-маска:** проста форма, потрібен чіткий край, економія файлів.
**Коли ffmpeg-alpha:** у відео м'які краї / світіння / часткова прозорість, які силуетом не передаси.

---

## JS — прогрів відео проти блимання в Safari

Прибирає короткий чорний/білий кадр при старті прозорого відео в Safari.

```js
function supportsWebMAlpha() {
  if (typeof MediaSource !== 'undefined') {
    return MediaSource.isTypeSupported('video/webm; codecs="vp09.00.10.08"');
  }
  return false;
}

document.querySelectorAll('.hero-key').forEach((el) => {
  const video = el.querySelector('video');

  // Pause on first play to decode the first frame without visibly playing
  let preventPlay = true;
  video.addEventListener('play', () => {
    if (!preventPlay) return;
    video.pause();
    preventPlay = false;
  });

  // Pick source: WEBM for non-Safari, MOV for Safari
  const source = document.createElement('source');
  const isSafari = /^((?!chrome|android).)*safari/i.test(navigator.userAgent);
  if (supportsWebMAlpha() && !isSafari) {
    source.src = 'path/to/video.webm';
    source.type = 'video/webm';
  } else {
    source.src = 'path/to/video.mov';
    source.type = 'video/quicktime';
  }
  video.appendChild(source);

  // Warm up: load + first-frame decode. Play whenever you want afterwards.
  video.load();
  video.play().catch(() => {});
});
```

> `<video>` має початково мати атрибут `autoplay`. Після прогріву можна запускати відтворення будь-коли — перший кадр уже відрендерений.

---

## Швидкий чеклист

- [ ] Дізнався розмір відео (`ffmpeg -i file`)
- [ ] Маска з Figma експортована у достатньому розмірі
- [ ] Зробив **WEBM** (VP9) для Chrome/Firefox
- [ ] Зробив **MOV** (HEVC + `hvc1`) для Safari
- [ ] Прибрав звук (`-an`), якщо не потрібен
- [ ] Перевірив прозорість у Safari І в Chrome
- [ ] Підключив JS-прогрів, якщо є блимання в Safari
