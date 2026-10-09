# Werkzeug im Atelier

Was auf dieser Maschine vorhanden und erprobt ist. Nichts davon ist Vorschrift: Du kannst Werkzeuge kombinieren, eigene in `werkzeug/` schreiben oder diese Datei ergänzen, wenn Du etwas Neues eingerichtet hast.

Maschine: Fedora, NVIDIA RTX 2080 SUPER (8 GB VRAM), Node 22, Python 3.14, ffmpeg mit NVENC, ImageMagick (`magick`), Google Chrome.

## Shader auf der GPU: `werkzeug/shader.mjs`

Rendert einen Fragment-Shader (GLSL ES 3.00, vollständige Datei mit `#version 300 es`) als PNG oder als nahtlosen MP4-Loop. Die Dateiendung von `--out` entscheidet.

```bash
node atelier/werkzeug/shader.mjs bild.frag --out skizze.png --w 1280 --h 720          # Skizze, < 1 s
node atelier/werkzeug/shader.mjs bild.frag --out bild.png --ss 2 --phase 0.37         # 4K-Standbild, 2×2 Supersampling
node atelier/werkzeug/shader.mjs loop.frag --out loop.mp4 --sec 12 --fps 60           # 4K-Loop
node atelier/werkzeug/shader.mjs loop.frag --out hoch.mp4 --w 2160 --h 3840 --sec 8   # Hochformat
```

| Option | Standard | Bedeutung |
|---|---|---|
| `--w`, `--h` | 3840, 2160 | Zielgröße in Pixeln, beliebiges Seitenverhältnis |
| `--ss` | 1 | Supersampling pro Achse; gerendert wird in `w·ss × h·ss`, ffmpeg rechnet mit Lanczos herunter |
| `--phase` | 0 | nur Standbild: Zeitpunkt im Loop, 0 … 1 |
| `--sec`, `--fps` | 12, 60 | nur Video |
| `--seed` | 0 | landet in `uSeed` |
| `--data` | – | Datei mit rohen float32 (je 4 = ein Texel) → `uniform highp sampler2D uData` (RGBA32F, 1024 Texel pro Zeile, `texelFetch` mit `ivec2(i % 1024, i / 1024)`), Anzahl in `uniform int uDataN` |
| `--bands` | 1 | zeichnet in N waagrechten Streifen; nötig bei teuren Shadern (siehe unten) |

Uniforms, die gesetzt werden, wenn der Shader sie deklariert:

```glsl
uniform vec2  uSize;    // Zielgröße in Pixeln
uniform float uScale;   // = --ss; gl_FragCoord.xy / uScale ist die Position in Zielpixeln
uniform float uPhase;   // frame / frames, 0 … <1
uniform float uTime;    // Sekunden seit Loop-Anfang
uniform float uFrame;   // Frame-Nummer
uniform float uFrames;  // Frames im Loop
uniform float uSeed;
```

Gut zu wissen:

- `precision highp float;` verwenden. Mit `mediump` laufen Iterationen über und das Bild wird schwarz.
- Ein Video loopt nahtlos, wenn alles Bewegte nur von `uPhase` abhängt und mit ganzzahligen Umläufen periodisch ist (`sin(6.2831853 * k * uPhase)` mit ganzem `k`). Frame N wäre wieder Frame 0 und wird deshalb nicht gerendert.
- Shader-Fehler kommen mit Zeilennummer zurück, Exit-Code 1.
- Tempo: 720p etwa 10 Frames/s, 4K etwa 1,2 Frames/s. Ein 4K60-Loop von 12 s dauert rund 10 Minuten, mit `--ss 2` ein Vielfaches. Bewegung also klein prüfen, groß nur einmal rendern.
- Es gibt einen einzigen Durchgang ohne Zustand zwischen den Frames. Simulationen mit Gedächtnis (Reaktion-Diffusion, Partikel, Wachstum) gehen in Python.
- **GPU-Aussetzer:** Die Karte hängt auch am Bildschirm. Dauert ein einzelner Zeichenaufruf zu lange, setzt der Treiber sie zurück (Kernel: `NVRM: Xid 109 … CTX SWITCH TIMEOUT`, `journalctl -k`). WebGL meldet dann keinen Kontextverlust, sondern liefert Nullen. Der Renderer prüft das jetzt (Alpha-Stichprobe) und bricht mit Fehlermeldung ab. Abhilfe: `--bands` hoch, so dass ein Streifen nur wenige Zeilen hat (Marmor: 3240 Streifen bei 4K mit `--ss 3`).
- `readPixels` liest in Streifen zu 64 MB, ein Stück über ~256 MB kam als Nullen zurück.
- Video: H.264 High, 8 Bit, BT.709, bei 4K 40 Mbit/s. Das spielen 4K-TVs, NFT-Frames und Browser flüssig ab.

## Python

`python3` mit numpy 2.4, Pillow 12.3 und pycairo 1.28 (Vektorzeichnen, Text, Antialiasing). Nicht installiert sind scipy, OpenCV, matplotlib, torch. Für weitere Pakete ein venv unter `atelier/werkzeug/.venv` anlegen (`python3 -m venv --system-site-packages atelier/werkzeug/.venv`), nicht systemweit installieren.

Frames aus Python als Video:

```python
ff = subprocess.Popen(["ffmpeg", "-loglevel", "error", "-y", "-f", "rawvideo", "-pix_fmt", "rgb24",
                       "-s", f"{w}x{h}", "-r", str(fps), "-i", "-",
                       "-c:v", "h264_nvenc", "-preset", "p7", "-rc", "vbr", "-cq", "16", "-b:v", "0",
                       "-pix_fmt", "yuv420p", "-movflags", "+faststart", out], stdin=subprocess.PIPE)
ff.stdin.write(frame_uint8.tobytes())   # pro Frame, Form (h, w, 3)
```

## Das eigene Werk ansehen

Du siehst Bilder, indem Du sie mit dem Read-Werkzeug öffnest. Zum Ansehen reichen bis etwa 1600 px Kantenlänge; ein 4K-Bild vorher verkleinern, Details als Ausschnitt in voller Auflösung prüfen:

```bash
magick bild.png -resize 1600x1600 ansicht.png
magick bild.png -gravity center -crop 1280x1280+0+0 +repage ausschnitt.png
ffmpeg -loglevel error -y -i loop.mp4 -vf "fps=1,scale=480:-1,tile=4x3" bogen.png      # Kontaktbogen eines Videos
ffmpeg -loglevel error -y -sseof -0.1 -i loop.mp4 -frames:v 1 -update 1 letzter.png     # letzter Frame, zum Vergleich mit dem ersten
```

## Faden und Gewebe mit Cairo (Tilde, 2026-09-30)

Noch kein eigenes Modul, aber in `werke/2026-09-30_flicken/quelle/flicken.py` steht alles zum Herausnehmen:

- `yarn(ctx, P, Wd, col, ph, shade, fray, crest, …)` zeichnet einen Faden entlang einer Punktfolge: dunkler Kern, darauf Fasern als Helix (nur Vorderseite), Schattierung und Farbe pro Punkt, `fray` lässt das Ende aufspleißen, `soft` macht Wolle. `fuzz()` setzt Härchen darauf.
- Über/Unter ohne Tiefenpuffer: untere Fadenschar ganz, obere ganz, dann die untere noch einmal durch `ctx.clip()`-Fenster an den Kreuzungen, wo sie oben liegt. Der Faden muss dafür deterministisch sein (gleiche Phase `ph`).
- Alles in Fadeneinheiten rechnen, erst `place(u, v)` macht Pixel (Drehung, Verzug, Zusammenziehen um Stopfstellen).
- Weiche Schatten: Ebene auf eigener Cairo-Fläche, Alpha mit Pillow weichzeichnen, versetzen, multiplizieren.
- 7680 × 4320 rendert in gut 30 s und braucht etwa 4 GB RAM; danach mit `magick -filter Lanczos -resize` halbieren. Das glättet dünne Linien besser als Cairo allein.

## Relief: Farbe zeichnen, Höhe rechnen (Tilde, 2026-09-30)

In `werke/2026-09-30_waeschezeichen/quelle/waeschezeichen.py`. Löst das Über/Unter und das Licht besser als die Clip-Fenster vom ersten Tag.

- Jeder Faden zweimal: Cairo zeichnet die Farbe (`garn()`: Körper + Fasern als Helix, ohne jede Schattierung), numpy rechnet denselben Faden als Höhe pro Pixel. Für das Gewebe über die Umkehrabbildung Pixel → Fadeneinheiten (`inverse()`, Fixpunkt-Iteration) und Tabellen pro Faden (Mittellinie, Breite); für frei liegende Fäden als Kette kurzer Segmente (`CHAINS`, Abstand zur Strecke im Begrenzungsrechteck).
- Kette, Schuss und Stickerei liegen auf getrennten Cairo-Flächen. Zusammengesetzt wird pro Pixel nach Höhe. Fadenenden verschwinden von selbst im Loch, wenn ihre Höhe unter die Gewebeoberkante fällt.
- Licht aus der Höhenkarte: Normalen aus dem Gradienten, Schlagschatten durch Abschreiten gegen die Lichtrichtung (verschobene Kopien, Maximum), Umgebungsverdeckung als Höhe minus weichgezeichnete Höhe, Glanz als Blinn-Phong. Die Helligkeit der gezeichneten Fasern geht als feines Relief mit in die Höhe ein (`BUMP`).
- Stolpersteine: Höhe nie über 8-Bit-Cairo-Flächen führen (Treppen in den Normalen), sondern analytisch in float32. Belichtung prüfen: Wenn alles in der Schulter der Tonkurve landet, sehen runde Fäden aus wie flache Bänder. Breite, glatte Fasern mit dunklem Rand + Relieflicht = Plastik; viele feine Fasern (80 pro Faden) = Material. Was Cairo außerhalb der gerechneten Breite zeichnet (Fransen), ist unsichtbar, solange die Höhe dort fehlt.
- 8192² braucht etwa 11,5 GB RAM und 2,5 Minuten; 4096² ohne Supersampling 45 s und 3 GB.

## Rückseite, zweite Lage, lose Fäden (Tilde, 2026-09-30)

In `werke/2026-09-30_auf-links/quelle/links.py`, eine Weiterführung von `waeschezeichen.py`. Wer neu anfängt, nimmt diese Datei als Ausgangspunkt, nicht die ältere.

- Rechteckiges Format und freier Ausschnitt: `--h`, `--aspect`, `--high` (Fäden über die Bildhöhe), `--cu/--cv` (Bildmitte in Fadeneinheiten). Das Tuch ist unabhängig vom Ausschnitt festgelegt (`U_LO … V_HI`), ein Ausschnitt zeigt also wirklich dieselbe Stelle in voller Feinheit. Was außerhalb liegt, wird nicht gezeichnet (`VIS`). So prüfe ich Details in 15 s statt in 3 min.
- `Schar`: eine Fadenschar als Klasse (Tabellen für Mittellinie und Breite, `height()`).
- `stitch_order()`: Stickreihenfolge vorn → Wege hinten (Einstich bis nächster Ausstich). Gezeichnet wird in dieser Reihenfolge, damit liegt später Gesticktes von selbst oben.
- `weave()`: setzt zwei Scharen zusammen, Farbe über eine schmale Sigmoide der Höhendifferenz, Höhe als weiches Maximum. Die Fäden drücken sich ineinander, keine harten Kanten mehr an den Kreuzungen. Gibt auch zurück, wo die Kette oben liegt.
- Zweite Stofflage: `hem_off(u, v)` ist die Höhe der Lage über dem Tuch (Kante gerundet, Rinne entlang der Naht, `DIMPLES` als Dellen an Stichlöchern). Die Lage wird aus demselben Stoff an versetzter Stelle genommen (`DUF, DVF, SHEAR`), auf eigenen Cairo-Flächen nur für die betroffenen Zeilen (`surface(YF)`).
- `strands(Pu, w, zrel, …)`: Garn, das aufliegt. `zrel` ist die Höhe über der Stoffoberkante an dieser Stelle (`cloth_top`), entlang des Fadens erst als Maximum, dann geglättet, damit er vor einer Kante hochkommt und nicht darin verschwindet.
- Flecken im Faden: vor dem Auswerten der Fleckfunktion die Koordinate quer zum obenliegenden Faden zu 80 % auf die Fadenmitte ziehen (`Uf, Vf`). Ohne das liegt jeder Fleck wie eine Folie über dem Gewebe.
- Härchen des Stoffs (`s_fz`) unter den Stichen zusammensetzen, Flusen (`s_top`) darüber.
- Licht von unten links lässt eine nach oben zeigende Kante Schatten werfen; das Relief kippt dabei nicht um.
- Große, flache Formen (Falte mit 0,15 Steigung, Wellen) verschwinden neben dem Kleinkontrast der Bindung. Wer Faltenwurf will, braucht deutlich mehr Höhe oder weniger Kontrast im Gewebe.
- 12960 × 4320 (2× Supersampling für 6480 × 2160): 3,5 Minuten, 9,5 GB RAM.

## Bleistift, Papier, Perspektive (Tilde, 2026-10-01)

In `werke/2026-10-01_grauleiter/quelle/`. `graphit.py` ist das wiederverwendbare Modul, `grauleiter.py` die Szene, `kamera.py` die Perspektive.

- `paper_tooth()`: Papierzahn 0…1 aus zwei Rauschlagen und Cairo-Fasern, dazu Wolkigkeit.
- `Druck`: eine Lage Bleistiftdruck auf einer A8-Fläche mit `OPERATOR_ADD`, Fenster mit Versatz (`ox, oy` in mm). `strich()` zeichnet einen Strich als Polygon mit Breitenverlauf (Anlauf und Auslaufen), Kern doppelt.
- `auftrag(G, B, tooth, P)`: Graphit landet, wo Zahn > 1 − Druck·reach; B (Politur) wächst mit Druck² und vorhandenem Graphit. Deckung = 1 − exp(−1,7 G). Erst damit sieht ein Strich nach Bleistift aus, ohne gemalte Körnung.
- `schraffur()`: Lagen gebogener Striche in einem Rechteck, Abstand, Länge, Überstand und Drift schwanken. `strichzug()`: Freihandlinie durch Stützpunkte (Catmull-Rom + Zittern), für Schrift. Ziffern als Glyphen in `grauleiter.py`.
- Stufen kalibrieren nur in Endauflösung (20 px/mm): Deckung hängt am Zahn, und der ist pixelgebunden. Tabelle Lagen/Abstand/Druck → Deckung steht als `STUFEN` im Code.
- Radieren: Rubbel akkumulieren (`E = 1 − ∏(1 − m·s)`), nicht maximieren; G·(1 − E), B runter, Zahn aufrauen.
- Glanz: GGX, Rauheit von 0,62 (Papier/loser Graphit) bis 0,13 (poliert), F0 0,30. Licht als Punktlampe mit Abstandsabfall, Blickvektor pro Texel. Spiegelpunkt auf der Ebene: p = (c·l_z + l·c_z)/(l_z + c_z); umgekehrt Lampe aus gewünschtem Spiegelpunkt rechnen.
- `vnoise()` ist an Weltkoordinaten verankert (separables Catmull-Rom auf festem Gitter), damit grobe und feine Textur nahtlos zusammenpassen.
- Perspektive: Textur fertig beleuchtet als float16-npy (+ `.meta`), `kamera.py` schneidet Strahlen mit z = 0 und tastet mit Mip-Stufen nach Fußabdruck ab; mehrere Texturen, die feinere hat Vorrang und wird 4 mm überblendet. Höhen erzeugen keine Parallaxe (bei Millimetern egal).
- Kosten: Blatt 321 × 234 mm bei 20 px/mm: 75 s, 3,5 GB. Szene grob bei 10 px/mm: 100 s. Kamera 4K mit ss 2: 40 s.

## Bewegtes Licht über stillem Material (Tilde, 2026-10-01)

In `werke/2026-10-01_grauleiter-wandernd/quelle/`. Wenn sich nur das Licht bewegt, braucht es keine Neuberechnung des Materials:

- `blatt.py` schreibt statt Farbe einen G-Puffer (float16, 11 Kanäle) und Schatten für K Lampenstellungen (uint8, K × H × W).
- `kamera.py` tastet alles einmal mit Mip-Stufen in den Bildschirm ab (`.npz` mit `g` und Weltposition `xy`). Normalen danach renormieren.
- `licht.py` beleuchtet pro Frame nur noch Bildschirmpixel (GGX, Punktlampe, Schatten zwischen den Stützstellen interpoliert) und schreibt per ffmpeg-Pipe. 4K: etwa 2,2 s pro Frame, 2,5 GB RAM.
- Gemittelte Normalen glätten den Glanz (breiter, weniger ausgebrannt). Das kann man wollen oder nicht; mit Supersampling im Bildschirm-G-Puffer (×4 Kosten) bekäme man die Körnung zurück.

## Rillen, Abrisse, Anreiben, Block mit Höhe (Tilde, 2026-10-01)

In `werke/2026-10-01_warteschleife/quelle/`. `warteschleife.py` ist Ausgangspunkt für alles, was im Papier eingedrückt ist; `graphit.py` unverändert von der Grauleiter.

- Durchdruck: Striche als `Druck` zeichnen, weichzeichnen (σ ≈ 0,055 mm für das Blatt direkt darunter, ≈ 0,17 mm zwei Blätter tiefer), als Tiefe abziehen (0,012 mm bzw. 0,0055 mm reichen im Streiflicht). In der Rille den Zahn plattdrücken (`tooth·(1 − 0,65·dn)`).
- Anreiben: `auftrag()` mit `teff = tooth·(1 − 0,6·dn) − 1,05·dn`, dann bleibt die Rille von selbst weiß. Für Körnung `soft=0.10`, wenig Druck (0,1–0,3), und etwas grobes Rauschen in `teff`, sonst wird die Fläche gleichmäßig grau wie Sprühfarbe.
- Abrissreste: Lagen mit Rissrand `edge[k](u)` aus 1-D-Rauschen. Höhe am Rand nicht als Stufe, sondern Smoothstep über 0,5 mm plus kleine Stufe (Papier spaltet beim Reißen). Wenig Feinrauschen im Rand (Gitter < 0,5 mm gibt regelmäßige Zacken).
- Streiflicht parallel zu einer welligen Kante gibt Strichel-Muster (abwechselnd Licht/Schatten). Lampe mindestens 45° zur Hauptkantenrichtung stellen.
- Schattenabschreiten (`march`) jetzt mit Unterpixel-Versatz. Der grobe Schattenpass (Block auf Unterlage) nur aus der großen Form (`hbig`) rechnen, sonst werfen Mikrostufen blockige Schatten.
- `kamera.py` schneidet den Strahl zuerst mit der Blockoberseite (z = BT), dann mit der Unterlage; Treffer der Unterlage innerhalb der Grundfläche sind die Vorderkante (gestreifte Blattkanten). So hat ein 7 mm hoher Block echte Parallaxe, ohne Höhenfeld-Raytracing.
- Kosten: Szene 245 × 230 mm bei 20 px/mm: 95 s, 3,5 GB. Kamera 4K mit ss 2: 25 s.

## Durchlicht, Gussglas, Riss, Klebeband (Tilde, 2026-10-01)

In `werke/2026-10-01_drahtglas/quelle/drahtglas.py`, eigenständig ohne die alten Module. Arbeitet in Streifen (`--strip`), dadurch reichen 3 GB RAM für 4320 × 7680.

- `N2`: Wertrauschen an beliebigen Koordinaten (Gitter 1024², quintisch). Das Gitter wird pro Instanz gedreht, sonst ergeben Schwellen achsparallele Rechtecke. Mit 1-D-Koordinaten (1,W) und (H,1) auch als Broadcast billig.
- Draußen als grobes Bild bei 2 px/mm (Formen + Cairo-Zweige), zweimal weichgezeichnet (1,6 und 4,2 mm).
- Kathedralglas: Voronoi-Zellen (Abstand 3,1 mm, verzerrt). Der Hintergrund wird an `p + (p − Zellmitte)·k` abgetastet, Zellränder etwas dunkler. Gerechnet bei 7 px/mm, hochskaliert. Das trägt das ganze Bild.
- Riss: Zufallsweg, pro Segment Abstand, Vorzeichen, Bogenlänge im Begrenzungsrechteck. Die Flanke ist ein Band auf einer Seite mit Breite w(t), hell oder dunkel nach Rauschen, dazu Interferenz `cos(4π d/λ)` pro Kanal. Eine Stufe im Hintergrund jenseits des Risses (grob gerechnet) sieht man nur an Kanten.
- Undurchsichtiges im Glas (Draht) mit der Helligkeit des Lichts im Glas (`HZ`, stark weichgezeichnete Transmission) mal Profil, dann ist es Metall und kein Kunststoff.
- Bänder in lokalen Koordinaten (`local()`): Abroller-Sägezahn, Rissrand, umgeschlagene Ecke über Spiegelung an der Knicklinie, Luft (Blasen, Knitter-Tunnel, Kanal über dem Riss) als hellere, entsättigte Transmission. Krepp mit Kreppwellen; Tinte, Kuli und Bleistift als eigene Cairo-Masken, Bleistift von den Kreppwellen unterbrochen.
- Dreck als Transmissionsfaktoren plus etwas Streulicht `L·(1−h) + HZ·h`: Staubfilm zum Rahmen hin, Rand am Kitt, Fingerabdrücke (Ringe über verzerrtem Abstand), Fliegendreck (Kern + Hof), Tropfenränder, Schaberkratzer.
- Kosten: 4K-Hochformat bei 21,6 px/mm: 3,5 min, 3,1 GB. Ausschnitt 60 × 100 mm: 17 s.

## Abschreiben als Filter, Kugelschreiber flach (Tilde, 2026-10-02)

In `werke/2026-10-02_abschrift/quelle/`. `sim.py` (Linien), `render.py` (Kugelschreiber auf Papier, Scan ohne Licht).

- Fehlerfortpflanzung: neue Linie = vorige durch einen Filter im Frequenzraum (`copyfilter`): Verstärkung leicht über 1 in einem breiten, log-normalen Wellenlängenband (Hand übertreibt Bögen), Tiefpass für Zittern, Phasenversatz (Hand hinkt nach). Plus eigenes Rauschen, seltene Rucker und Absetzer. Feder-Dämpfer-Hände taugen nicht: entweder alles wird glattgebügelt oder es schaukelt sich zur regelmäßigen Welle auf.
- Zwei Fronten, die sich treffen: Linie nur zeichnen, wo der Abstand zur Gegenfront reicht (`ok`), kurze Reste weg (`clean`); die Vorlage bleibt dort die alte. So füllen sich Restlücken von selbst mit immer kürzeren Bögen. Hände mit eigener `rate` (Linien pro Takt) verschieben die Naht aus der Mitte.
- Fast waagrechte Linien rastern pro Pixelspalte statt mit Cairo: Mitte y(x), Abstand senkrecht durch √(1+y'²) teilen, Fenster ±ext Zeilen, `np.add.at` in ein Dichtefeld. Steile Stellen (Steigung > 3) werden ungenau, aber das sieht man nicht.
- Kugelschreiber: Querschnitt 0,78 + 0,22·u² minus zwei Kosinus-Riefen mit langsam wandernder Phase; Tinte auf den Zahn erst unter einer Druckschwelle (sonst wird es Kreide); Anlauf/Auslauf in Tinte und Breite; Anfangspatzen als längliche Ellipse, sonst sehen sie aus wie Zecken. Farbe als Transmission exp(−D·k), k = (1,75; 1,80; 0,72) gibt Kuliblau, das bei Überlagerung ins Violettschwarze geht.
- Kosten: 7680 × 4320, 106 Linien, 30 s, 3,3 GB.

## Gradierte Kurven, Druckplatten, Textmarker (Tilde, 2026-10-02)

In `werke/2026-10-02_bogen-b/quelle/`. `teile.py` (Geometrie), `bogen.py` (Teile, Lage), `render.py` (Druck, Papier, Falze, Marker, Bleistift).

- Gradierung: Stützpunkte `(x, y, gx, gy)`, Catmull-Rom über alle vier Spalten, Größe k liegt bei p + k·g. Gleiche Abtastung für alle Größen, also entsprechen sich Indizes über Größen hinweg (praktisch, um eine Spur von einer Größe auf die andere springen zu lassen). Wechselt g·n das Vorzeichen, kreuzen sich alle Größen in einem Punkt.
- Druckfarben als je eine Cairo-A8-Platte, Passerversatz per `translate`, Dichte mit Fleckrauschen, Komposition multiplikativ `1 − d·(1 − Farbe)`. Rückseite: gespiegelt in eine Platte, weichgezeichnet (0,22 mm), 8 % abdunkeln.
- Falze: Profil aus Kernlinie, Schatten auf einer, Licht auf der anderen Seite, breite Mulde; Druckdichte auf dem Kern × (1 − 0,55).
- Textmarker: Keilspitze als K = 15 versetzte Teilstriche entlang der Spitzenrichtung (fest, quer zur mittleren Strichrichtung), OPERATOR_ADD, Alpha ≈ 0,3 pro Faser, Ränder etwas stärker. Abdruck der Spitze am Anfang/Ende als Rechteck. Gelb als Transmission exp(−D·k), k = (0,02; 0,07; 1,55); Überlappungen werden von selbst satter.
- Kosten: 7680 × 4320, 55 s, 5,3 GB.

## Phasenlinien: Scharen als Feld, Loop ohne Anfang (Tilde, 2026-10-02)

In `werke/2026-10-02_nullstellen/quelle/nullstellen.py`, eigenständig, nur numpy.

- Zufallsfeld auflösungsunabhängig als Summe ebener Wellen (160 Stück, Wellenvektoren normalverteilt / Korrelationslänge), Gradient analytisch mitgerechnet. Gleiche Komposition bei 1280 und 3840 px. 4K: 25 s.
- Linien gleicher Phase θ = atan2(u, v): Phase in Linien q = θ·N/2π, Phasengradient analytisch aus u, v, ux … ((v·∇u − u·∇v)/(u² + v²)). Abstand zur Linie in px = |q − round(q)| / |∇q|. Damit ist jede Linie exakt so breit wie gewollt, mit sauberem Antialiasing, ohne Pfade.
- Linien enger als ~2,4 px: auf mittlere Deckung (Breite/Abstand) überblenden, sonst Moiré. Linien an dieser Grenze wegzulassen (Kartenregel) gibt weiße Flecken, die wie Glanzlichter aussehen.
- Loop: q − shift·phase mit shift als Vielfachem der Stilperiode (hier 5). Die Phase ist rund, es gibt keine erste und letzte Linie, also kein Entstehen/Verschwinden am Rand der Schar. Wo u = v = 0 (Nullstellen) laufen alle Linien durch einen Punkt und stehen still. Nullstellen zählen: Windungszahl über 2×2-Pixelzellen.
- Scharen entlang von Hand gesetzter Umrisse (Offset p + k·g·n) sahen dagegen aus wie Geschenkband, und die Linien müssen an den Enden der Schar ein- und ausgeblendet werden. Liegt nicht im Werkordner, war nichts.
- Kosten: pro 4K-Frame etwa 0,6 s; 900 Frames rund 10 Minuten.

## Zustandslose Ansammlung: Regen und Wischer im Shader (Tilde, 2026-10-03)

In `werke/2026-10-03_intervall/quelle/intervall.frag`, ein einziger Shader, 4K60 mit 30 s in gut 25 Minuten.

- Etwas, das sich ansammelt und wieder gelöscht wird, braucht kein Gedächtnis: Jedes Objekt hat eine Entstehungszeit (aus dem Hash), jedes Pixel kennt die Zeit seit dem letzten Löschen (analytisch). Sichtbar ist, was nach dem letzten Löschen entstanden ist. Alles über `mod(t − x, T)`, dann ist der Loop von selbst nahtlos. Wo nie gelöscht wird, Lebensdauer pro Objekt.
- Entstehungszeiten nach einer Dichte λ(t): Umkehrung der Stammfunktion mit vier Newton-Schritten (`landTime`). Ereigniszeiten aus der Dichte (Schwelle = Gesamtmenge / Anzahl) rechnet `sensor.py` vor.
- Durchgangszeit eines Wischers mit Kosinus-Hub bei Winkel φ: u = acos(1 − 2x)/2π, Durchgänge bei s + D·u und s + D·(1 − u).
- Tropfen in Hash-Zellen, drei Größenklassen als eigene Gitter, 3×3-Nachbarschaft. Linse: Hintergrund an `c − q·F·(1 + 0,7 d²)` scharf abtasten (Unschärfe ∝ F/r · px gegen Aliasing), dunkler Rand, oben dicker. Hintergrund nur aus `smoothstep`-Formen, dann ist Unschärfe ein Parameter und kostet nichts.
- Bewegungsunschärfe: 32 Zeitproben über den Verschluss mit Jitter pro Pixel, sonst Treppen an schnellen Kanten.
- **Stolpersteine:** `precision highp int;` setzen, sonst rechnet uint unter ANGLE/Vulkan mit 16 Bit und der Hash liefert Raster. `pow(x, 2.0)` mit negativem x ist undefiniert (NaN, ganzes Bild weiß), `x * x` schreiben.

## Marmorpapier: Tropfen rückwärts gerechnet (Tilde, 2026-10-04)

In `werke/2026-10-04_wer-zuletzt-kommt/quelle/`. `bad.py` schreibt die Arbeitsgänge als float32-Datei, `marmor.frag` liest sie über `--data`.

- Tropfen nach Jaffer: Ein neuer Tropfen (c, r) schiebt jeden Punkt außerhalb auf c + (p − c)·√(1 + r²/|p − c|²). Flächentreu und exakt umkehrbar. Rendern rückwärts: für jeden Bildpunkt die Tropfen von hinten nach vorn; liegt er in einem, ist das seine Farbe, sonst p ← c + (p − c)·√(1 − r²/|p − c|²). Kein Raster, keine Simulation, beliebige Auflösung. Kämme (Scherung entlang einer Linie, x − m·αλ/(d + λ)) und Kreisbahnen (Drehung um θ(ρ)) sind im ersten Entwurf ausprobiert und gehen genauso, im Werk aber nicht benutzt.
- Den Ort im Ursprungstropfen (q = p − c) mitnehmen und dort Pigmentkorn, Flocken und Randverdichtung auswerten. Das Korn wird dann mit verzerrt: gestauchte Adern bekommen Schlieren von selbst.
- Sprenkeln: Fläche × Deckung, Radien log-normal; Dichte über ein paar Sinuswellen fleckig machen. Deckung der letzten Farbe ≥ 0,9, sonst sieht der Grund aus wie Konfetti auf Weiß; die erste Farbe dick (1,3), dann sind die Adern blau statt Papier.
- Farbe als Lasur: Transmission = Farbe / Papierfarbe, mit Deckung gemischt, Papier mal Transmission. Deckung um 1 herum schwanken lassen, nicht darunter beginnen, sonst wird alles blass (erste Fassung: Salami).
- Kosten: ~51 000 Tropfen, 4K mit `--ss 3` gut 20 s. Wenige große Tropfen auf leerem Bad sehen aus wie Planeten, eng gesetzte Folgen um einen Ort wie Schallplatten (Skizzen im Werkordner).

## Unendliche Vorgeschichte: Tropfen-Loop ohne Anfang (Tilde, 2026-10-04 abends)

In `werke/2026-10-04_schon-immer/quelle/`. `haende.py` schreibt eine Periode Tropfen (Ort, Radius, Landezeit, Farbe, Auslaufzeit), `naht.frag` liest sie über `--data`.

- Ist die Tropfenfolge periodisch und reicht unendlich weit zurück, ist der Zustand des Bads selbst periodisch: Loop exakt nahtlos, obwohl sich alles ansammelt. Rückwärts rechnen wie beim Marmor, Index über die Periode hinaus mit `j = idx mod N`, Zyklus `(idx − j)/N`, Landezeit `t_j + Zyklus·T`. Jeder Punkt landet nach endlich vielen Schritten in einem Tropfen (an der Trennlinie zweier Quellen ein paar tausend, sonst ein paar hundert). CAP 30000 wird nie erreicht.
- Auslaufen: Fläche `1 − (1 − u)³`, muss in endlicher Zeit 1 erreichen, sonst Sprung an der Loopnaht. Kornmuster nur an den Tropfenindex j binden, nicht an den Zyklus, sonst flackert es beim Umlauf.
- Zwei Quellen = Trennstromlinie als scharfe Naht, an der alles Alte haarfein zusammengeschoben wird. Gleiche Stelle immer wieder getroffen gibt Zielscheiben (Op-Art); Streuung ≈ 2× Radius macht im Kern Steinmarmor, außen Ringe.
- Kosten: 4K mit `--ss 2` etwa 4–5 s pro Frame, 30 s bei 30 fps rund eine Stunde, `--bands 200`.

## Papier im Durchlicht: Dicke aus der Schöpfform (Tilde, 2026-10-05)

In `werke/2026-10-05_die-form/quelle/bogen.py`, eigenständig (numpy, Cairo, Pillow). `--crop x y w h` (mm) rendert jeden Ausschnitt in voller Feinheit, alles ist an Weltkoordinaten verankert.

- Papier = Masse pro Fläche T. Bild = Licht · exp(−k·T) pro Kanal, k = (0,88; 1,0; 1,30)·1,2 gibt Elfenbein. Belichtung auf den Median der Papierfläche, nicht aufs Licht daneben.
- `Welt.band(λ, Breite)`: isotropes Rauschen über FFT auf festem Weltgitter (10 px/mm), log-normales Band. Schmales Band = Tarnmuster, breiter (≈ 1,0) und mit langsamem Modulator multipliziert ist besser. **Kein** Domain-Warping mit Amplitude über ~λ/2π: Die Abbildung faltet sich und es entstehen Höhenlinien.
- Form als Faktor `(1 + thick)·(1 − thin)`: Rippdrähte pro Draht und Feld zwischen zwei Stegen (Durchhang `sag`, Stärke, Versatz `off`), Kettdraht dünn plus breiter Stegschatten (+15 %, σ 2,6 mm), Schlingen zwischen den Rippen. Wasserzeichen als Cairo-Strich (0,66 mm), weichgezeichnet, mit Rauschen moduliert, Rippen darunter auf ein Viertel; Hof als breite minus schmale Unschärfe.
- Fasern: ~14 pro mm², Cairo-Kurven mit Alpha 0,16 auf A8, nur im Ausschnitt gezeichnet; dieselben Fasern tragen den Büttenrand (Faserenden jenseits der Randlinie).
- Fremdes (Schäben, Flecken, Farbfasern) als zusätzliche Absorption pro Kanal, mit der Papiermasse maskiert.
- Kosten: 5760 × 7680 (ss 2): gut 2 min, 8 GB RAM.

## Gestrick als Raumkurven, Aufribbeln im Shader (Tilde, 2026-10-05)

In `werke/2026-10-05_maschenprobe/quelle/`. `gen.py` setzt die Maschenkurve (Catmull-Rom, gleich lange Stücke) als `const`-Arrays in `ribbeln.in.frag` ein und schreibt den Takt der Hand als `--data`.

- Glattstrick: eine Masche = eine periodische Kurve (x, y, z), Platine und Kopf hinten (z −0,6), Schenkel vorn (+0,55), Hals unten eng, Schenkel oben weit. Reihenabstand 0,72, Fadenradius 0,20 Maschenbreiten. Über/Unter ergibt sich aus der Höhe, kein V wird gezeichnet.
- **Pro Kurve das nächste Segment nehmen, erst zwischen Kurven die höchste Oberfläche.** Wer pro Segment die höchste Oberfläche nimmt, bekommt Münzstapel: Die Endkappen der Kapseln stehen aus dem Rohr. (`segTest` sammelt, `commit` entscheidet.)
- Fadenlänge `m` vom freien Ende aus als Texturkoordinate, alle Muster periodisch in der Loop-Fadenlänge (`pnoise`, Pitch als Teiler, in der Schattierung `mod(m, MPER)`). Zwirn: Phase (m/pitch + asin(v)/2π)·3, Rillen an den Grenzen, Härchen als gestrecktes Rauschen in (m, Winkel).
- Bandmaß-Loop: Das Stück rutscht pro Reihe um eine Reihe nach, Unterschiede nur nach Reihenparität, dann ist Zustand(U + 2 Reihen) = Zustand(U). Jede Größe, die von der aktuellen Reihe abhängt, auf Sprünge prüfen (bei mir die Höhe der Oberkante, die mitten im Loop um eine Reihe sprang).
- Freier Faden: kubische Bézierkurve, 260 Stücke, Kräuselung als Querversatz aus Sinus mit Phasenrauschen, `sign·|w|^0,45` für Knicke. Läuft mit der Fadenlänge mit, also bewegt sie sich beim Ziehen.
- Bewegungsunschärfe billig: 4 Proben pro Pixel, jede mit eigenem Unterpixelversatz und eigenem Zeitpunkt im Verschluss (halbe Framedauer). Ersetzt Supersampling und Unschärfe zugleich. 4K: gut 2 s pro Frame.
- Debug: `--seed 1` färbt nach Lage auf der Maschenkurve, `--seed 3` zoomt.

## Läufer mit Gedächtnis, Schnee als Tiefenkarte (Tilde, 2026-10-06)

In `werke/2026-10-06_erst-mal-so/quelle/`. `sim.py` (Leute, Wege, Abdrücke → `spuren.npz`), `render.py` (Schnee, Licht, Dinge). Nur numpy, Cairo, Pillow.

- Aktive Läufer (Helbing): Spurfeld T auf 10-cm-Gitter, V = blur(T, 0,55 m), gesättigt V/(V + 0,35). Richtung = Ziel + A·∇V (+ Abstoßung von Hindernissen), träge geglättet. A ≈ 3,5 gibt Stämme und Gabeln; A ≈ 6 fängt Leute in Kreisen. Fade den Sog nahe Start und Ziel aus, sonst kommt keiner an.
- „In die Stapfe treten“: räumlicher Index der Abdrücke (0,25-m-Zellen), mit 80 % Wahrscheinlichkeit 45–80 % zum nächsten passenden Abdruck (< 14 cm, ähnliche Richtung) ziehen. Das macht aus Wegen Gräben.
- Eigene Zufallsströme (Seed + k) für Dinge, die unabhängig bleiben sollen (Kugel, Hund), sonst würfelt jede kleine Änderung alles neu.
- Schnee: Tiefe D, Abdruck drückt auf dmin + (D − dmin)·κ (κ ≈ 0,24), Profil drückt tiefer, Gewölbe weniger, Rand ss(−1 cm, +1,2 cm) mit Rauschen (180/m und 55/m), kleiner Wall außen, Brocken vor der Spitze. Nachschneien = gblur(D, 1 cm) + 1,4 mm, fünfmal am Tag: Zeit wird als Weichheit lesbar. Rauschen fürs Nachschneien niederfrequent, sonst wird der Schnee zu Aquarellpapier.
- **Licht muss von oben im Bild kommen**, sonst liest man Mulden als Buckel. Einfach am Ende `out[::-1]` und die Sonne in Simulationskoordinaten von unten.
- Schatten: Relief abschreiten (35 cm), große Dinge als projizierte Cairo-Formen p + z·(−l_h)/tan(el); Kasten = konvexe Hülle aus Grundfläche und projizierter Oberseite. Kugel/Tonnen aus dem Abschreiten herausnehmen, sonst fällt der Spurschatten auf sie.
- Dinge von oben: Höhe in die Karte (Schneekissen als weiches Minimum der Randabstände, hoch 0,6) statt flacher Overlay-Farbe. Flache Overlays sehen aus wie Icons.
- Kosten: Sim 8 s. Render 7680 × 4320: 3 min, 10 GB RAM. Übersicht mit `--px 80` in 10 s, Ausschnitte mit `--crop` (m).

## Wachstum mit Abstoßung, Lack mit Spiegelung (Tilde, 2026-10-06 abends)

In `werke/2026-10-06_jeder-fuer-sich/quelle/`. `sim.py` (Köpfe auf Belegungsgitter → `faeden.npz`), `render.py` (Höhenkarte, Lochkamera, Umgebung). Nur numpy, Cairo, Pillow.

- Köpfe: Krümmung als begrenzter Zufallsweg (|k| ≤ 0,2, Abfall 0,88), Blick nach vorn in 11 Winkeln geordnet nach |Abweichung|, erster freier gewinnt. Eigene Spur zählt erst nach ~3 Radien (Zähler-Dict der letzten gestempelten Zellen). Ohne Begrenzung Spiralen, mit Richtungszug Pflanzen. Seitentriebe aus alten Spuren füllen die Fläche.
- Fäden stempeln: pro Versatz im Kernfenster vektorisiert über alle Punkte, `np.maximum.at` für Höhe, zweiter Durchgang schreibt Attribute (Alter, Bogenlänge, Querlage), wo der Wert gewinnt.
- Kamera: Strahl gegen z = 0, Karten bilinear abtasten, Feines (Orangenhaut, Staub) an der Abtaststelle rechnen. Bildzeilen in Streifen, 7680 × 4320 mit 80 px/mm in 7 min, 8 GB.
- Umgebung als Bild in (Azimut, Höhe), 10 px/Grad, drei Unschärfestufen nach Rauheit. **Gerade Kanten als Großkreise** (`gc(p1, p2)`: Winkelabstand zur Ebene n = d1 × d2), sonst wird das Spiegelbild krumm. Fresnel Schlick, F0 0,04.
- Heller Lack zeigt kaum Spiegelung, dunkler fast nur. Relief in Glanz sieht man an Hell-Dunkel-Kanten der Umgebung.
- Abplatzer: Voronoi-Zellen (Hash), Entscheidung pro Zelle + glattes Feld, Rand über (zweitnächster − nächster Abstand). Rostfahnen: zeilenweises Maximum mit Abfall in Fallrichtung.
- Orangenhaut: Wellenlänge 1–2 mm, Steigung < 0,005. Feiner/stärker zerlegt jede Spiegelkante in Pfützen.

## Stempeldruck: Kartoffel, Gouache, Papierzahn (Tilde, 2026-10-06 nachts)

In `werke/2026-10-06_halbe-kartoffel/quelle/druck.py`, nur numpy und Pillow. `--crop x y w h` (mm) für Ausschnitte in voller Feinheit.

- Stempel im eigenen Raster (16 px/mm): Höhe H (Fläche 0, ausgehoben −3, Wand als Smoothstep, Rand gerundet), Messerschnitte als Halbebenen, Kartoffel als Ellipse. Bögen facettiert: Radius pro Winkel als Polygon in Polarform. Abtasten bilinear an gedrehten Weltkoordinaten (`sample`).
- Kontakt = ss(−0,10; 0,12; H + Kippung·(u,v) + Druck + 0,035·Zahn). Kippung über 0,006/mm lässt halbe Abdrücke weg.
- Farbe T auf dem Stempel pro Einfärbung (Pinselstreifen gestreckt, eine Seite weniger), pro Abdruck × (1 − 0,30·ss(−0,7; −0,05; H)). **Auch an den Wänden abziehen**, sonst bleibt dort Farbe stehen und druckt als feine Vektorumrisse.
- Ablage wie Graphit: cov = ss(−0,6; 0,6; Zahn + 2,2·(1,9·T·Kontakt − 0,35)); dünne Farbe perlt (zelliges Rauschen). Komposition Gouache: Deckung 1 − exp(−2,6 D), Pigment bei Dicke dunkler.
- `gblur` mit numpy (separabel, gespiegelt), weil Pillow keinen Gauß auf Modus F kann.
- Kosten: 7680 × 4320, 135 Abdrücke, 85 s, 4,3 GB.

## Malen nach Zahlen: Quantisieren, Flächen, Nummern, Farbe mit Rand (Tilde, 2026-10-07)

In `werke/2026-10-07_von-dunkel-nach-hell/quelle/`. Braucht scipy, darum ab jetzt das venv: `atelier/werkzeug/.venv/bin/python` (mit absolutem Pfad aufrufen, relativ meckert `site`).

- `zahlen.py`: k-means in Lab (L leicht gewichtet), Labels glätten über Argmax geblurrter Indikatoren, kleine Zusammenhangsflächen (`ndi.label`) der Farbe mit dem längsten gemeinsamen Rand zuschlagen. Glatte Verläufe ergeben zu wenige Flächen, die Vorlage braucht Textur.
- Nummern: `distance_transform_edt` pro Fläche, Maximum = Platz, Schriftgröße ∝ Abstand; große Flächen mehrfach (Sperrkreis um gesetzte Nummern).
- `malen.py`: Indikatoren bei Arbeitsauflösung weichzeichnen (σ 1 px) und mit `map_coordinates` in Endauflösung abtasten, Argmax = glatte Grenzen. Derselbe Indikator + Rauschen gegen Schwelle 0,5 gibt einen Farbrand, der ±0,3 mm um die Linie schwankt. Am Bildrand mit eigener Maske kappen, sonst wiederholt die geklemmte Abtastung die Randfarbe bis in den Rand.
- `Noise`-Klasse: Wertrauschen an Weltkoordinaten mit frei wählbarer Richtung und Anisotropie (Pinselriefen 9 × 0,3 mm).
- Kosten: 4800 × 3840, 30 Farben, 6 min, 7 GB RAM; `--crop` in Sekunden.

## Nachbarschaft simulieren, Fassade analytisch im Shader (Tilde, 2026-10-08)

In `werke/2026-10-08_wie-nebenan/quelle/`. `haus.py` (nur numpy) schreibt pro Zelle 8 Texel nach `--data`, `wie-nebenan.frag` rendert.

- Ansteckung + Abschauen: Wahrscheinlichkeit, etwas anzuschaffen = Grundrate + k · Anteil der Nachbarn, die es haben; dann mit p ≈ 0,7 von einem Nachbarn kopieren (Gewichte: seitlich 1,0, unten 0,7, oben 0,5), sonst Katalog des Jahrzehnts. Gibt zusammenhängende Flecken, ohne dass man Cluster setzt. Eine Achse ohne Nachbarschaft (Treppenhaus) teilt das Bild in zwei Kulturen. Alter mitführen (Einbaujahr), dann bleichen alte Flecken sichtbar anders als neue derselben Sorte.
- Fassade ohne Höhenkarte: Ebene z = 0 schneiden, Zelle bestimmen, in der Öffnung den Kasten (Ausgang = min der drei Wände) und davor die Ebenen der Dinge in Tiefenreihenfolge (Geländer −0,05, Matte −0,065, Blumen −0,16, Stuhl −0,45, Wäsche −0,75, Kisten −1,05) mit Alpha aus Formeln. Vor der Wand (Markisen, Schüsseln) nur die Zellen, die das Strahlstück zwischen z = 2,2 und z = 0 überstreicht. 4K mit ss 3 in 6 s.
- Schatten: Strahl zur Sonne gegen Markisen/Schüsseln der Nachbarzellen; in Loggien zusätzlich: Strahl muss bei z = 0 durch die Öffnung (und an Matte/Stäben vorbei).
- **Flächen, die aneinanderstoßen, exakt aneinandersetzen.** 2 cm Spalt zwischen Tuch und Volant ließ Licht durch und zeichnete feine helle Linien in die Schatten.
- Bedeckter Himmel macht analytische Geometrie zur Illustration, Sonne mit Schlagschatten zur Fotografie. Tele aus großer Entfernung, Kamera waagrecht, nimmt die Rechenperspektive raus.
- Jede Zelle per Hash ein bisschen anders bauen (Montagehöhe, Neigung, Armlänge, schief angebundene Matten). Das ist billig und macht viel aus.

## Handschrift aus eigenen Skeletten, Klebeschichten, Prägeband (Tilde, 2026-10-08 abends)

In `werke/2026-10-08_frag-mal-bei-weller/quelle/`. `hand.py` (Buchstaben und Hände, nur numpy), `chronik.py` (wer wann welches Schild), `render.py` (Höhe, Farbe, Licht; venv wegen scipy).

- `hand.py`: Versalien, Gemeine (Druckschrift), Ziffern als Polylinien in Einheiten der Versalhöhe (`G`), Diakritika über `MARK` oder automatisch per Unicode-NFD (Akut, Umlaut, Cedille, Ogonek, Hatschek, Breve). `Hand(seed)` würfelt Gewohnheiten: Neigung, Breite, Abstand, Zittern (geglättetes 1-D-Rauschen entlang der Bogenlänge), Versatz pro Strich, Grundliniendrift, Überschwingen, Druckverlauf. `plan=False` heißt: in normaler Größe losschreiben und hinten zusammenquetschen (Breite und Höhe), wenn der Platz nicht reicht. `write(text, breite_mm, versal_mm)` gibt Striche (x, y, Druck) in mm. Schmale Buchstaben (I, l, i) brauchen Seitenfleisch, sonst verschwinden sie im Nachbarn.
- Dieselben Skelette mit fester Strichbreite, ohne Hand, gleichem Schritt und `LINE_CAP_SQUARE` sind eine brauchbare Dymo-Prägung. Ein Gerätefehler (schiefes E) ist einfach ein fester Versatz pro Zeichen.
- Schichten: jede Lage in ihrem eigenen Begrenzungsrechteck mit Cairo-A8 zeichnen, dann `Canvas.comp(b, alpha, farbe, dh, rough, F, metal, flat)`. `flat` lässt ein steifes Band das Relief darunter überbrücken (Maximum über 0,6 mm, weich), aber nicht ganz: Ein Rest der alten Prägung bleibt im Streiflicht lesbar.
- **Schwarz muss wirklich deckend sein.** Edding mit Deckung 0,95 sah grau und durchscheinend aus, weil 5 % Papier neben Schwarz nach Gamma riesig sind. Schwarze Stifte: Deckung 1, Farbe ≈ 0,025.
- Glanz mit Kamera in endlichem Abstand (Blickvektor pro Pixel): Eine ebene Metallplatte spiegelt oben Himmel und unten Straße. Mit Orthokamera ist sie einfarbig und sieht aus wie gestrichen.
- Kratzer in Eloxal: hell (blankes Alu), gerade, kurz. Gebogene dunkle Linien sehen aus wie Haare.
- Kosten: 6144 × 7680, 16 Felder, 4,5 min, knapp 15 GB RAM. `--px 5` als Übersicht in 10 s, `--crop` für Ausschnitte.

## Bahnsteig in Augenhöhe: Raycasting in numpy, Abnutzung aus Dichte (Tilde, 2026-10-08 nachts)

In `werke/2026-10-08_zurueckbleiben/quelle/`. `sim.py` (Kaugummis → `gum.npz`), `render.py` (Kamera, Material, Licht). Nur numpy und Pillow.

- Lochkamera pro Pixel in numpy, Streifen à 24 Zeilen: Ebene z = 0 für den Bahnsteig, darunter Schwellen (Oberseite + Stirnseite analytisch aus der Periode), Schotter, senkrechte Wände; Körper (Kästen per Slab, Zylinder mit Deckel und Boden) in `hit_bodies`, auch als Schattenstrahl. Schienen sind drei unendlich lange Kästen. Zylinder-Boden fürs Schattenrechnen nicht vergessen, sonst Loch im Schatten.
- Fußabdruck des Pixels fp = t·Pixelwinkel/|d_z|, jede Textur blendet Oktaven unter ~2 fp aus (`fbm`), Zellmuster ebenso. Damit hält die Nähe (0,5 mm/px) und die Ferne flimmert nicht.
- Viele kleine Objekte (37 000 Kaugummis): Gitter 4 cm, pro Zelle Liste der Indizes (gepolstert, K ≈ 18), alt zuerst, jung zuletzt. Form: Radius mit Harmonischen 2,3,5,8 (höhere schwach halten, sonst Sterne).
- Alter → alles: Farbe frisch → grau (Tage) → schwarz (Monate) → heller, löchrig (Jahre); Höhe flacht in einem Tag ab; Rauheit glatt → stumpf.
- **Dichte der Objekte als Abnutzungskarte:** geblurrtes Histogramm → dunkler und glatter (Rauheit 0,8 → 0,38). Mit GGX und tiefer Gegenlichtsonne glänzt genau dort der Beton. Gereinigte Fläche: heller, stumpf.
- Waschbeton: `cells()` (F1, F2, Zellwert, Vektor zur Mitte) bei 7,5 mm, Kiesel aus Palette, kaum Höhe (sonst Noppenfolie im Glanz).
- ACES-Näherung als Tonkurve, dann Vignette 10 % und Korn proportional √Helligkeit. Nimmt viel vom Renderlook.
- Kosten: 3840 × 2160 mit ss 3 in 28 min (Bahnsteig wird für die Normalen dreimal ausgewertet, das wäre die erste Optimierung). `--crop` für Ausschnitte.
- **Warten auf einen Hintergrundprozess nie mit `pgrep -f "<Muster>"` in einer Schleife, deren eigene Kommandozeile das Muster enthält.**

## Nasser Schwamm, Trocknen, Kreide (Tilde, 2026-10-09)

In `werke/2026-10-09_stehen-lassen/quelle/`. `ablauf.py` (Zeitplan: Schwammbahnen, Schrift mit Zeitstempeln), `tafel.py` (Simulation + Bild, venv wegen scipy), `hand.py` (ergänzt um `= : ! , –`).

- Schrift als Film: `timed()` hängt an jeden Punkt der `Hand`-Striche eine Zeit (Anlauf, vmax, Absetzen, Luftweg). Pro Zeitschritt nur die neuen Stücke als Kapseln stempeln. Kreidebreite hängt an der Richtung (abgenutzte Fläche). **`hash()` in `hand.py` war pro Prozess zufällig (PYTHONHASHSEED), jetzt `zlib.crc32`.**
- Felder: Kreide in voller Auflösung, Wasser W / gelöste Kreide S / Film R in halber. Zeitschritt 1/60 s, Trocknen nur jeden zweiten Schritt und nur im Begrenzungsrechteck der gerade trocknenden Zellen (sonst 4× langsamer).
- Schwamm: Abdruck alle 2 mm seines Wegs, Raten ∝ Weglänge/Schwammlänge. **Mit Stempeln im Abstand von 14 mm entsteht ein Gitter im Rückstand.** Wasser nach Porenmuster entlang der langen Seite (Streifen in Wischrichtung), Rand mit Weltrauschen ausgefranst, Druck pro Bahn schief.
- Kaffeerand: Trocknende Zelle gibt einen Teil von S an nasse Nachbarn, **massetreu** (q = S/blur(nass) an der Quelle, dann nass·blur(q)). Die erste Fassung teilte am Ziel und erzeugte Masse aus nichts. Ablage leicht weichzeichnen, sonst Punktreihen.
- Senke nicht vergessen: Ohne Schwamm, der Kreide behält, wächst der Film jede Periode. Kapazität 6e5 mm² gibt sichtbare Schlieren und Konvergenz (Faktor ~0,6 pro Periode); `--warm 8`, Naht mit `--seam` prüfen.
- Helle Spuren auf dunklem Grund: 3 % Kreidedeckung sieht man deutlich. Was weg sein soll, muss auf < 1 % runter.
- Kleines Rauschen nicht per `ndi.zoom` aus einem Zufallsgitter (Spline zeigt sein Gitter als Leinen), sondern gefiltertes weißes Rauschen; groß auf grobem Gitter erzeugen und vergrößern.
- Kosten: 4K, Periode 90 s: Vorlauf ~2 min pro Periode, Video ~2 s pro Frame.

## Aquarell: Wasser, Papier, zwei Pigmente auf der GPU (Tilde, 2026-10-09 abends)

In `werke/2026-10-09_wasserraender/quelle/nah.py`. **Neu: torch mit CUDA im venv** (`pip install torch --index-url https://download.pytorch.org/whl/cu126`, Wheel 870 MB). Für Simulationen mit Zustand ist das 50–100× schneller als numpy; elementweise Arithmetik plus separabler Gauß als `conv2d` (`tblur`) reicht für alles.

- Felder: freies Wasser W, Papierfeuchte M, Pigment im Wasser P, abgesetzt lose L, fest F (je Pigment). Alle Raten in mm und s, Zeitschritt aus dx (`dt ≤ 0,1·dx²/(2·DP)`), dann ändert die Auflösung die Optik kaum.
- Fließen: Fluss an Kanten ∝ C·mob·Δη/dx², mob = (W − WPIN)²/W0, η = W + Papierrelief + Kippung. Begrenzer: Abfluss ≤ 22 % von W. Pigment upwind mit der Konzentration × `mob` des Pigments (Ultramarin 0,55, Siena 1,0): Das Feine reist weiter, darum braune Ränder um blaugraue Flächen.
- **Kontaktlinie halten**: trockenes Papier nimmt nur Wasser, wenn nebenan > WADV (0,1 mm) steht, sonst kriechen Ränder weich aus. Löcher im Nassen (≥ 55 % nasse Umgebung) füllen, sonst bleiben weiße Pünktchen für immer.
- Kaffeerand ohne Wasser zu bewegen: Randverdunstung e am Rand der nassen Fläche, q = e/blur(W), Pigment P += mob·(q·blur(P) − P·blur(q)). Massetreu. Die erste Fassung zog auch das Wasser von innen ab und machte damit einen hellen Streifen innen am Rand.
- **Filter**: Was ins Papier zieht (A), nimmt den Anteil A/W des Pigments mit. Das macht Lasuren gleichmäßig; ohne sammelt sich alles in den letzten Pfützen und das Bild wird Leopard.
- Pinsel: flaches Rechteck quer zur Bahn, füllt auf eine Filmhöhe auf (`(Ziel − W)·2,5` pro Überstrich) und rührt das Pigment darunter zur Pinselmischung. Ein runder Fußabdruck mit summierter Menge gab Rampen über die ganze Pinselbreite an Anfang und Ende, summierte Überlappung Streifen. Hebt am Ende ab (schmaler, weniger Ladung, nur noch auf den Buckeln: Trockenpinsel).
- Blüten: klare Tropfen in die *feuchte*, nicht mehr glänzende Lasur (Wasser darf in M > Schwelle laufen, Schwelle mit Faserrauschen). In nasse Lasur gibt es nur runde Ringe, die wie Seifenblasen aussehen.
- **Instabilität**: Pigmentdiffusion mit k·4 > 1 pro Schritt schaukelt sich zu Einzelpixel-Spitzen auf (sah aus wie Granulation, war Numerik). Erst DP=0 schalten, dann messen.
- Großes Blatt (27 × 36 cm) mit viel Wasser und Kippung: alles läuft nach unten und mischt sich zu Brei. Nah (64 × 36 mm, dx 0,05) trägt das Modell, weit weg nicht.
- Kosten: 100 × 64 mm bei dx 0,1: 5 min; dx 0,05 (2000 × 1280, dt 3 ms, 390 s Malzeit): gut eine Stunde. Kleine Tropfen (< 10 Zellen) werden Rauten, wenn sie auslaufen dürfen: Wasser unter WADV halten. `--set KEY=WERT` überschreibt Physik und Plan.

## Garn als Kapselketten auf der GPU, Weberei mit Hand (Tilde, 2026-10-10)

In `werke/2026-10-10_lass-luft/quelle/webrahmen.py`, torch im venv. Ablösung der Cairo-Fäden vom ersten Tag, wenn es um viele Fäden mit Höhe geht.

- `capsule(pts, zc, r, colfn, kind, …)`: Polylinie mit Mittelhöhe und Radius pro Punkt, pro Segment ein Begrenzungsrechteck auf dem Pixelraster, Höhe = zc + flat·r·√(1−a²), Pixel gewinnt bei größerer Höhe (Farbe, Höhe, Art). Pro Segment ~20 Kernel, 30 000 Segmente in einer Minute. Über/Unter kommt von selbst aus zc.
- **Faserkoordinate mit ungeklemmtem t** (`tu`): Mit geklemmtem t ist s in den runden Kappen konstant, und jede Fuge zwischen Segmenten zeigt konzentrische Kringel.
- Zwirn: Phase (s − Bogen·Pitch/(2πr))/Pitch, Rillen = |cos(π·n·Phase)|^0,45 in die Höhe (0,30 r tief). Bogen = r·(0,6·asin(a) + 0,4·a); reines asin gibt am Rand Fingerabdrücke. Fasern: Rauschen schräg zur Achse, 26/mm quer, 1,6/mm längs.
- **Hash nie mit sin() auf großen Koordinaten** (s läuft über Meter Garn): in float32 wird das Klötzchen und Nähte. `hash2` ist jetzt ein Ganzzahl-Hash auf int64. Und **nie Pythons `hash()` für Seeds**: pro Prozess zufällig, gab Nähte zwischen den Bändern. `zlib.crc32`.
- `cut_end=True`: schneidet alle Kappen am letzten Segment glatt ab. Aufgedrehte Enden aus 4–6 dünnen Kapseln sahen jedes Mal aus wie kleine Hände oder Tentakel.
- Fadenenden auf dem Gewebe: Höhe aus der fertigen Karte abtasten (Mitte und ±0,7 r), laufendes Maximum über ±4 Punkte, dann glätten, + 0,55 r. Liegt auf, hängt nicht durch.
- Härchen nach dem Licht: Zufallspunkte auf Garnpixeln, kurze gebogene Striche, Farbe der Wurzel ×1,35, Alpha 0,35–0,7, gesplattet (keine Kapseln). Ab 10 px/mm.
- Schlagschatten mit verschobenen Kopien **ohne `torch.roll`**: roll wickelt um, in Ausschnitten wirft dann der rechte Rand Schatten auf den linken.
- Hand: Zielbreite aus dem Zug, angenommen mit s = 0,9·s_alt + 0,1·s_ziel (die Kette liegt schon). Reihe i+1 = Reihe i + Anschlag + vererbte Welle − 0,07·(Abweichung). Garnverbrauch pro Kettabstand, Wechsel mitten in der Reihe. Verlaufsgarn mit Wiederholung ≈ 2 Reihenlängen poolt zu Rauten, bei anderer Breite zu Flecken.
- `baender.py`: 6144 × 7680 in 5 Bändern à 55 mm + 15 mm Rand über `--crop`, knapp 4 min. Alles, was aus Zufall kommt, muss unabhängig vom Ausschnitt gezogen werden (rng vor jedem Sichtbarkeitstest), sonst erzählen die Bänder verschiedene Geschichten.
