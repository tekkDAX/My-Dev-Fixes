Anleitung: Qwen-Code auf Termux installieren (Der definitive Guide)

Veröffentlicht von: [Dein Name/Pseudonym hier]

Einleitung

Das KI-Coding-Tool qwen-code ist eine leistungsstarke Erweiterung für Entwickler, aber es wurde primär für Standard-Desktop-Betriebssysteme wie Linux, Windows und macOS entwickelt. Eine offizielle Unterstützung für Termux auf Android existiert bisher nicht. Eine Standardinstallation mit npm install -g @qwen-code/qwen-code schlägt daher mit einer Kaskade von Fehlern fehl.

Diese Anleitung ist das Ergebnis einer intensiven Debugging-Session und zeigt einen zuverlässigen Weg, qwen-code auf Termux zu portieren und lauffähig zu machen. Wir werden dabei kritische Abhängigkeiten patchen, den Build-Prozess korrigieren und die Software an die Besonderheiten von Termux anpassen.

Voraussetzungen

Stelle sicher, dass alle notwendigen Build-Werkzeuge in Termux installiert sind. Falls nicht, führe folgenden Befehl aus:

code
Bash
download
content_copy
expand_less
pkg update && pkg upgrade
pkg install nodejs git python build-essential
Schritt 1: Den Quellcode herunterladen

Wir klonen das offizielle Repository in unser Home-Verzeichnis. Alle folgenden Schritte finden in diesem Verzeichnis statt.

code
Bash
download
content_copy
expand_less
cd ~
git clone https://github.com/qwen-team/qwen-code.git
cd qwen-code
Schritt 2: patch-package zur Reparatur vorbereiten

Da eine Abhängigkeit (@lvce-editor/ripgrep) fehlerhaft ist, benötigen wir patch-package, um eine permanente Reparatur durchzuführen.

Installiere patch-package als Entwicklungsabhängigkeit:

code
Bash
download
content_copy
expand_less
npm install patch-package --save-dev

Öffne nun die package.json-Datei mit nano package.json und füge im "scripts"-Block eine postinstall-Anweisung hinzu. Achte darauf, dass die Kommas korrekt gesetzt sind (alle Zeilen außer der letzten im Block müssen mit einem Komma enden).

code
JSON
download
content_copy
expand_less
// In deiner package.json:
"scripts": {
  "prepare": "npm run bundle",
  "prepare:package": "node scripts/prepare-package.js",
  // ... andere Skripte ...
  "clean": "node scripts/clean.js",
  "postinstall": "patch-package"  // <-- Diese Zeile hinzufügen (ohne Komma am Ende!)
},
Schritt 3: Den Patch erstellen (Der Kern der Reparatur)

Jetzt führen wir eine Installation durch, die die fehlerhaften Skripte ignoriert. Dadurch erhalten wir den node_modules-Ordner, den wir reparieren können.

code
Bash
download
content_copy
expand_less
npm install --ignore-scripts

Nun öffnen wir die fehlerhafte Datei. Sie kann leer sein oder fehlerhaften Code enthalten – das spielt keine Rolle.

code
Bash
download
content_copy
expand_less
nano node_modules/@lvce-editor/ripgrep/src/downloadRipGrep.js

Lösche den gesamten Inhalt dieser Datei und ersetze ihn durch den folgenden, vollständig korrigierten Code. Dieser Code behebt das "android"-Plattformproblem und entfernt Abhängigkeiten zu nicht existierenden Paketen.

code
JavaScript
download
content_copy
expand_less
// Kompletter Inhalt für downloadRipGrep.js
import { VError } from '@lvce-editor/verror';
import { execa } from 'execa';
import extractZip from 'extract-zip';
import fsExtra from 'fs-extra';
import got from 'got';
import os from 'node:os';
import { createWriteStream } from 'node:fs';
import { dirname, join } from 'node:path';
import { pipeline } from 'node:stream/promises';
import { randomUUID } from 'node:crypto';

const REPOSITORY = 'BurntSushi/ripgrep';
const VERSION = '13.0.0';

const temporaryFile = () => {
  return join(os.tmpdir(), `ripgrep-temp-${randomUUID()}`);
};

const getTarget = () => {
  let platform = os.platform().toLowerCase();
  if (platform === 'android') {
    platform = 'linux';
  }
  const arch = os.arch();
  switch (platform) {
    case 'darwin':
      return arch === 'arm64' ? 'aarch64-apple-darwin.tar.gz' : 'x86_64-apple-darwin.tar.gz';
    case 'win32':
      switch (arch) {
        case 'x64': return 'x86_64-pc-windows-msvc.zip';
        case 'arm64': return 'aarch64-pc-windows-msvc.zip';
        default: return 'i686-pc-windows-msvc.zip';
      }
    case 'linux':
      switch (arch) {
        case 'x64': return 'x86_64-unknown-linux-musl.tar.gz';
        case 'arm': case 'armv7l': return 'arm-unknown-linux-gnueabihf.tar.gz';
        case 'arm64': return 'aarch64-unknown-linux-gnu.tar.gz';
        case 'ppc64': return 'powerpc64le-unknown-linux-gnu.tar.gz';
        case 's390x': return 's390x-unknown-linux-gnu.tar.gz';
        default: return 'i686-unknown-linux-musl.tar.gz';
      }
    default:
      throw new VError(`Unknown platform: ${platform}`);
  }
};

const downloadFile = async (url, outFile) => {
  try {
    const tmpFile = temporaryFile();
    await pipeline(got.stream(url), createWriteStream(tmpFile));
    await fsExtra.mkdirp(dirname(outFile));
    await fsExtra.move(tmpFile, outFile);
  } catch (error) {
    throw new VError(error, `Failed to download "${url}"`);
  }
};

const unzip = async (inFile, outDir) => {
  try {
    await fsExtra.mkdirp(outDir);
    await extractZip(inFile, { dir: outDir });
  } catch (error) {
    throw new VError(error, `Failed to unzip "${inFile}"`);
  }
};

const untarGz = async (inFile, outDir) => {
  try {
    await fsExtra.mkdirp(outDir);
    await execa('tar', ['-xzf', inFile, '-C', outDir]);
  } catch (error) {
    throw new VError(error, `Failed to extract "${inFile}"`);
  }
};

export const downloadRipGrep = async () => {
  const target = getTarget();
  const url = `https://github.com/${REPOSITORY}/releases/download/${VERSION}/ripgrep-${VERSION}-${target}`;
  const homeDir = os.homedir();
  const downloadPath = join(homeDir, '.cache', 'ripgrep', `ripgrep-${VERSION}-${target}`);
  const binPath = join(homeDir, '.local', 'bin');
  const rgPath = join(binPath, 'rg');

  if (await fsExtra.pathExists(rgPath)) {
    console.log('ripgrep is already installed.');
    return;
  }

  if (!(await fsExtra.pathExists(downloadPath))) {
    await downloadFile(url, downloadPath);
  }

  if (downloadPath.endsWith('.tar.gz')) {
    await untarGz(downloadPath, binPath);
  } else if (downloadPath.endsWith('.zip')) {
    await unzip(downloadPath, binPath);
  }
  
  if (await fsExtra.pathExists(rgPath)) {
    await fsExtra.chmod(rgPath, '755');
  }
};

Speichere die Datei. Jetzt sichern wir diese Änderung als Patch:

code
Bash
download
content_copy
expand_less
npx patch-package @lvce-editor/ripgrep
Schritt 4: Die Installation abschließen und das Projekt bauen

Führe npm install ein letztes Mal aus. Dieses Mal wird unser Patch automatisch angewendet und die Installation läuft erfolgreich durch, inklusive des Build-Prozesses.

code
Bash
download
content_copy
expand_less
npm install
Schritt 5: Die reparierte Version global installieren

Wir installieren jetzt unsere lokal reparierte und gebaute Version global.

code
Bash
download
content_copy
expand_less
npm install -g .
Schritt 6: Die Startdatei manuell korrigieren

Der Build-Prozess erstellt eine fehlerhafte Startdatei. Wir ersetzen sie durch eine funktionierende Version.

code
Bash
download
content_copy
expand_less
nano /data/data/com.termux/files/usr/bin/qwen

Lösche den gesamten Inhalt und ersetze ihn durch diese zwei Zeilen:

code
Bash
download
content_copy
expand_less
#!/usr/bin/env node
import('/data/data/com.termux/files/usr/lib/node_modules/@qwen-code/qwen-code/bundle/gemini.js');

Speichere die Datei. Zum Schluss müssen wir sie noch ausführbar machen:

code
Bash
download
content_copy
expand_less
chmod +x /data/data/com.termux/files/usr/bin/qwen
Schritt 7: Verifizierung

Die Installation ist abgeschlossen. Teste sie mit:

code
Bash
download
content_copy
expand_less
qwen --version

Du solltest die Versionsnummer (z.B. 0.0.14) sehen. Herzlichen Glückwunsch, qwen-code läuft jetzt auf Termux
