# redrum.js | &lt; 5kb Markdown parser with external link and CSS class support

## Introduktion

En enkel och effektiv parser på under 5kb icke minifierad med funktionalitet för externa länkar och CSS klasser. Parsern hämtar varje markdown dokument, omvandlar innehållet till HTML och lägger resultatet i `<div>`-elementet.

Spara som en fil, exempelvis `parser.js`. Exempel på HTML:

```html
<div data-md-src="dokument.md"></div>
<script src="parser.js"></script>
```

**Parsern tillåter HTML-taggar i Markdown** och lägger resultatet i **`innerHTML`**. Använd därför bara Markdown från en betrodd källa eller sanera med exempelvis `DOMPurify`.

### 1. Parserns struktur och inline-regler

```js
const MarkdownParser = {
  inlineRules: [
    // spara HTML-taggar och kommentarer tillfälligt som token
    { reg: /<!--[\s\S]*?-->|<\/?[a-zA-Z][^>]*>/g, replace: (m, t) => `\u0000T${t.push(m) - 1}\u0000` },

    // bilder: ![alt-text](bildadress)
    { reg: /!\[([^\]]*)\]\(([^)\s]+)\)/gi, replace: '<img src="$2" alt="$1">' },

    // länkar: [text](adress) eller [text](ext adress)
    { reg: /\[([^\]]+)\]\(((?:ext\s+)?([^)\s]+))\)/gi,
      replace: (_, l, f, u) =>
        `<a href="${u}"${f.toLowerCase().startsWith('ext ') ? ' target="_blank"' : ''} rel="noopener noreferrer">${l}</a>` },

    // genomstruken text: ~~text~~
    { reg: /~~([^~]+?)~~/g, replace: '<del>$1</del>' },

    // fet text: **text**
    { reg: /\*\*([^*]+?)\*\*/g, replace: '<strong>$1</strong>' },

    // understruken text: __text__
    { reg: /__([^_]+?)__/g, replace: '<u>$1</u>' },

    // Kursiv text: *text*
    { rkg: /(^|[^\w])\*([^* \n][^*\n]*?)\*(?!\w)/g, replace: '$1<em>$2</em>' },

    // text med CSS klass: :klass text:
    { reg: /:([a-zA-Z0-9_-]+)\s+([^:]+?):/g, replace: '<span class="$1">$2</span>' }
  ],
```

### 2. start av `parse` och tillstånd

```js
  parse(md) {
    // html samlar den färdiga HTML-koden och paragraphs samlar rader som ska bli ett stycke
    const tokens = [], html = [], paragraphs = [];

    // parsern håller reda på om den befinner sig i en tabell eller lista
    let inTable = false, inList = false, listType = null;
```

### 3. Funktionen som avslutar ett stycke

```js
    const flush = () => {
      if (!paragraphs.length) return;

      const text = paragraphs.join("\n").trim();

      if (text) {
        // börjar stycket med :klass skapas ett p-element med klass
        const m = text.match(/^:([a-zA-Z0-9_-]+)\s+(.+)$/s);
        html.push(
          m
            ? `<p class="${m[1]}">${this.parseInline(m[2], tokens)}</p>`
            : `<p>${this.parseInline(text, tokens)}</p>`
        );
      }
      paragraphs.length = 0;
    };
```

### 4. Funktionen som stänger en lista

```js
    const closeList = () => {
      if (inList) {
        html.push(`</${listType}>`);
        inList = false;
        listType = null;
      }
    };
```

### 5. Markdown-rader och tabellhantering

```js
    // normalisera radslut och gå igenom dokumentet rad för rad
    for (const line of md.replace(/\r\n?/g, "\n").split("\n")) {
      const trimmed = line.trim();

      // rad som börjar och slutar med | behandlas som en tabellrad
      if (trimmed.startsWith("|") && trimmed.endsWith("|")) {
        closeList();

        // öppna tabellen första gången en tabellrad hittas
        if (!inTable) {
          flush();
          html.push("<table>");
          inTable = true;
        }

        // hoppa över en avgränsningsrad, exempelvis |---|---|
        if (/^\|[\s\d\w]*[:\s\-|]*\|$/.test(trimmed) && trimmed.includes("-")) {
          continue;
        }

        const cells = trimmed.split("|").slice(1, -1);
        const tag = html[html.length - 1] === "<table>" ? "th" : "td";

        html.push(
          `<tr>${cells.map(c =>
            `<${tag}>${this.parseInline(c.trim(), tokens)}</${tag}>`
          ).join("")}</tr>`
        );

        continue;
      }
```

### 6. Listor

```js
      // punktlistor och numrerade listor
      const listMatch = trimmed.match(/^([\*\-\+]|\d+\.)\s+(.+)$/);

      if (listMatch) {
        const isOl = /\d/.test(listMatch[1]);
        const type = isOl ? "ol" : "ul";
        const content = listMatch[2];

        // ändras listtypen stängs den gamla listan först
        if (inList && listType !== type) closeList();

        if (!inList) {
          flush();
          html.push(`<${type}>`);
          inList = true;
          listType = type;
        }

        html.push(`<li>${this.parseInline(content, tokens)}</li>`);
        continue;
      }
```

### 7. Avslut av tabeller, listor och tomma rader

```js
      // om nästa rad inte är en tabellrad avslutas tabellen
      if (inTable) {
        html.push("</table>");
        inTable = false;
      }

      closeList();

      // tom rad avslutar det aktuella stycket.
      if (!trimmed) {
        flush();
        continue;
      }
```

### 8. HTML, bilder och rubriker på egna rader

```js
      // HTML skickas vidare utan att göras om till stycken
      if (/^<\/?[a-zA-Z][^>]*>|^<!--[\s\S]*-->/.test(trimmed)) {
        flush();
        html.push(line);
        continue;
      }

      // ensam bildrad renderas utan ett omslutande p-element
      if (/^!\[([^\]]*)\]\(([^)\s]+)\)$/.test(trimmed)) {
        flush();
        html.push(this.parseInline(trimmed, tokens));
        continue;
      }

      // # till ###### motsvarar h1 till h6
      const h = line.match(/^(#{1,6})[ \t]+(.+?)\s*#*\s*$/);

      if (h) {
        flush();
        html.push(
          `<h${h[1].length}>${this.parseInline(h[2], tokens)}</h${h[1].length}>`
        );
        continue;
      }

      // rader som inte styrs av en annan regel blir ett stycke
      paragraphs.push(line);
    }
```

### 9. Avslut av dokumentet och inline-parsning

```js
    // stäng element som fortfarande är öppna
    if (inTable) html.push("</table>");
    closeList();
    flush();

    const result = html.join("\n");

    // töm tokens efter parsningen och returnera HTML-resultatet
    tokens.length = 0;
    return result;
  },

  parseInline(text, tokens) {
    // spara HTML-taggar tillfälligt så att vanliga tecken i dem inte escapas
    let res = text.replace(
      this.inlineRules[0].reg,
      m => this.inlineRules[0].replace(m, tokens)
    );

    // escape <, > och &
    res = res.replace(/[<>&]/g, c => ({
      "<": "&lt;",
      ">": "&gt;",
      "&": "&amp;"
    }[c]));

    // kör övriga inline-regler i ordning
    for (let i = 1; i < this.inlineRules.length; i++) {
      res = res.replace(this.inlineRules[i].reg, this.inlineRules[i].replace);
    }

    // lägg tillbaka HTML som sparades som tokens
    return res.replace(/\u0000T(\d+)\u0000/g, (_, i) => tokens[i]);
  },
```

### 10. Hämtning av Markdown och automatisk initiering

```js
  async init() {
    const controller = new AbortController();

    // avbryt pågående hämtningar när sidan lämnas
    window.addEventListener("pagehide", () => controller.abort(), { once: true });
    window.addEventListener("unload", () => controller.abort(), { once: true });

    try {
      // hitta alla element med attributet data-md-src
      await Promise.all(
        Array.from(document.querySelectorAll("[data-md-src]")).map(async div => {
          const url = div.dataset.mdSrc;

          try {
            const startFetch = performance.now();

            // hämta Markdown dokumentet
            const r = await fetch(url, { signal: controller.signal });
            if (!r.ok) throw new Error(`HTTP ${r.status}`);

            const text = await r.text();
            const endFetch = performance.now();

            // parsa Markdown till HTML
            const startParse = performance.now();
            const html = this.parse(text);
            div.innerHTML = html;
            const endParse = performance.now();

            // skriv ut tider för hämtning och parsning som log
            console.log(
              `%c${url}%c\n` +
              `Hämta: ${(endFetch - startFetch).toFixed(2)}ms\n` +
              `Parsning: ${(endParse - startParse).toFixed(2)}ms\n` +
              `Total Tid: ${(endParse - startFetch).toFixed(2)}ms`,
              "color: #007acc; font-weight: bold;",
              "color: inherit; font-weight: normal;"
            );
          } catch (e) {
            // visa fel i elementet, men ignorera avbrutna hämtningar
            // ändra från span samt klass vid behov
			if (e.name !== "AbortError") {
              div.innerHTML = `<span class="r">${e.message}</span>`;
            }
          }
        })
      );
    } catch (e) {
      console.error("redrum.js Init Error:", e);
    }
  }
};

// kör init då dokumentet laddats
document.readyState === "loading"
  ? document.addEventListener("DOMContentLoaded", () => MarkdownParser.init())
  : MarkdownParser.init();
```
