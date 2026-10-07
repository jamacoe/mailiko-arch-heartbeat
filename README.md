# mailiko-arch Heartbeat

Öffentlicher Beweiswert-Anker eines privaten, revisionssicheren Archivs nach
**BSI TR-03125 (TR-ESOR)**, Modul TR-ESOR-M.3, umgesetzt mit **mailiko-arch**.

Der Archivinhalt bleibt privat. Veröffentlicht wird stündlich nur, was sich
nachträglich nicht mehr ändern lässt, ohne dass es auffällt:

| Pfad | Inhalt |
|---|---|
| `heartbeat/YYYY/MM/DD/HHmmssZ.json` | ein Heartbeat: Anzahl und Gesamtgröße der Archivobjekte, Zahl der nach Ablauf der Aufbewahrungsfrist ausgesonderten Objekte (`archive.disposed`), Befunde der Integritätsprüfung, Zustand der Beweiskette |
| `heartbeat/chain.log` | fortlaufende Kette: `seq  utc  sha256(datei)  sha256(vorgänger)  pfad` |
| `heartbeat/latest.json` | Kopie des jüngsten Heartbeats |
| `mailiko-arch/roots.jsonl` | je Zeile ein Archivzeitstempel: Wurzel des Merkle-Hashbaums, TSA-Zeit, Seriennummer |
| `mailiko-arch/ats/<runId>.tsr` | das zugehörige RFC-3161-Zeitstempel-Token |

Keine Dateinamen, keine Inhalte, keine personenbezogenen Daten.

## Prüfen

### Mit mailiko-arch, in einem Schritt

Die Heartbeat-Prüfung von mailiko-arch prüft die lokale Arbeitskopie, mit `-Remote` einen
frischen Klon dieses Repositories und mit `-Deep` zusätzlich das vollständig nachgerechnete Archiv.

Sie macht automatisch, was unten von Hand steht: Kette nachrechnen,
Commit-Signaturen prüfen, die veröffentlichten Zeitstempel gegen das Archiv
halten. Jede Abweichung wird einzeln benannt — welcher Eintrag, welche Datei,
welcher Lauf. Rückgabewert `0` ohne Befund, `1` bei Befunden, `2` wenn nicht
prüfbar. `-Remote` prüft, was GitHub Dritten tatsächlich ausliefert, statt der
Arbeitskopie auf dem eigenen Rechner.

### Von Hand, ohne mailiko-arch

Die Kette der Heartbeats nachrechnen:

```bash
while read -r seq utc digest prev path; do
  actual=$(sha256sum "$path" | cut -d' ' -f1)
  [ "$actual" = "$digest" ] || echo "Abweichung in $path"
done < heartbeat/chain.log
```

Jeder Heartbeat nennt im Feld `prev` den SHA-256 seines Vorgängers. Ein
nachträglich entfernter oder geänderter Eintrag bricht damit jeden späteren
Eintrag und jeden darüber liegenden signierten Commit.

Sinkt `archive.objects` von einem Heartbeat zum nächsten, muss `archive.disposed`
um mindestens denselben Betrag steigen: Dann wurden Objekte nach Ablauf ihrer
Aufbewahrungsfrist geordnet gelöscht. Jede andere Abnahme ist ein Befund.

Ein Archivzeitstempel lässt sich einzeln prüfen:

```bash
openssl ts -reply -in mailiko-arch/ats/<runId>.tsr -token_in -text
```

Das Feld `messageImprint` muss der `root` derselben Zeile in
`mailiko-arch/roots.jsonl` entsprechen. Diese Wurzel deckt alle Datenobjekte des
Laufes ab; welches Objekt darunterhängt, weist der Inhaber mit dem jeweiligen
Evidence Record nach (RFC 4998), ohne die übrigen offenlegen zu müssen.

## Signaturen

Alle Commits sind mit einem SSH-Schlüssel signiert. Lokal prüfbar mit:

```bash
git log --show-signature
```

Der erwartete Schlüssel steht in `.allowed_signers`.
