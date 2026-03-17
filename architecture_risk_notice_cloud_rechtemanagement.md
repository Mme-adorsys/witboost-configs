# Architecture Risk Notice: Cloud-Rechtemanagement – Skalierungsrisiko bei zunehmender Cloud-Adoption

| | |
|---|---|
| **Dokument-Typ** | Architecture Risk Notice |
| **ID** | ARN-IAM-001 |
| **Version** | 0.1 |
| **Datum** | 17.03.2026 |
| **Status** | DRAFT – zur Diskussion |
| **Kontext** | DATEV Cloud-Migration / Handlungsfeld IAM & Berechtigungsmanagement |

---

## 1. Zusammenfassung

Das neue Cloud-Rechtemanagement ist als zentraler Policy Decision Point (PDP) für die DATEV Cloud-Infrastruktur konzipiert. Die aktuelle Implementierung – ein einzelner Endpunkt der binäre Berechtigungsentscheidungen (true/false) zurückliefert – ist für die Lastdimension, die sich durch die fortschreitende Cloud-Migration ergibt, architektonisch nicht ausreichend skalierbar.

Dieses Dokument beschreibt das strukturelle Risiko, quantifiziert die Lastproblematik und skizziert Richtungen, die eine nachhaltige Architektur ermöglichen würden. Es ist als sachlicher Beitrag zur Architekturdiskussion zu verstehen – das beschriebene Risiko entsteht nicht durch Fehler in der bisherigen Implementierung, sondern durch das Wachstum der Anforderungen im Zuge der Cloud-Migration.

---

## 2. Ausgangslage

### 2.1 Aktuelle Implementierung

Das Cloud-Rechtemanagement basiert auf einer Spring-Boot-Anwendung mit Neo4J als Backend. Es modelliert die Berechtigungsstrukturen von DATEV in einem komplexen Graphen und stellt einen einzelnen Abfrageendpunkt bereit:

- **Eingabe:** Benutzeridentität + angefragte Ressource / Aktion
- **Ausgabe:** `true` oder `false`

Um die aktuelle Last von ca. **30 Millionen Anfragen pro Tag** zu bewältigen, wird ein **Decision-Cache mit 30 Minuten TTL** eingesetzt. Das bedeutet: Eine einmal getroffene Berechtigungsentscheidung gilt für 30 Minuten als gültig – unabhängig davon, ob sich die Berechtigung in diesem Zeitraum geändert hat.

### 2.2 Einordnung in die ABAC/XACML-Terminologie

Um die Problemstellung präzise diskutieren zu können, ist folgende Begriffsdefinition hilfreich:

| Begriff | Abkürzung | Funktion |
|---|---|---|
| Policy Administration Point | PAP | Wo Berechtigungsregeln definiert und verwaltet werden |
| Policy Decision Point | PDP | Wo Berechtigungsentscheidungen getroffen werden |
| Policy Enforcement Point | PEP | Wo Entscheidungen technisch durchgesetzt werden (vor dem Ressourcenzugriff) |
| Policy Information Point | PIP | Wo Kontextinformationen für Entscheidungen bezogen werden |

In der aktuellen Implementierung übernimmt das Cloud-Rechtemanagement PAP, PDP und PIP zentral in einem System. Ein dedizierter PEP existiert nicht – die Durchsetzung obliegt den konsumierenden Anwendungen.

---

## 3. Das Skalierungsrisiko

### 3.1 Lastprojektion

Die aktuelle Last von 30 Millionen Anfragen pro Tag entsteht durch eine vergleichsweise kleine Anzahl bestehender Cloud-Anwendungen. Mit fortschreitender Cloud-Migration ändert sich diese Dimension grundlegend:

| Ebene | Anzahl | Basis |
|---|---|---|
| DATEV-Mitglieder (Steuerberater, Kanzleien) | ~40.000 | Schätzwert |
| Mandanten je Mitglied (Durchschnitt) | ~100 | Schätzwert |
| Mitarbeiter je Kanzlei | variabel | – |
| Datensatzarten / Berechtigungsobjekte | ~1.500 | DATEV-intern |

Allein die Kombination Mitglied × Mandant ergibt **4 Millionen Rechtsentitäten**. Werden Mitarbeiter und Datensatzarten einbezogen, bewegt sich die Anzahl möglicher Berechtigungskombinationen im **Milliardenbereich**.

Sobald Anwendungen wie Data-Next oder weitere Cloud-native Plattformen das Cloud-Rechtemanagement als Berechtigungsquelle nutzen, ist ein Anstieg von 30 Millionen auf **mehrere Milliarden Anfragen pro Tag** realistisch – eine Größenordnung von Faktor 100 oder mehr.

### 3.2 Schwachstellen der aktuellen Architektur unter dieser Last

**Problem 1 – Single Point of Truth als Single Point of Failure:**
Ein zentraler Endpunkt, der für jede Berechtigungsentscheidung synchron abgefragt werden muss, wird unter dieser Last zum Flaschenhals. Jede Latenz oder Nichtverfügbarkeit des Systems blockiert alle abhängigen Anwendungen.

**Problem 2 – 30-Minuten-Cache als Sicherheitsrisiko:**
Der Cache löst das Lastproblem nur scheinbar. Berechtigungsänderungen (z.B. Entzug von Zugriffsrechten) wirken erst nach Ablauf der Cache-TTL. In sicherheitskritischen Kontexten – etwa bei Kündigung eines Mitarbeiters oder Entzug einer Vollmacht – ist ein 30-minütiges Fenster nicht akzeptabel.

**Problem 3 – Binäre Antwort verhindert lokale Policy-Enforcement:**
Da der Endpunkt ausschließlich `true` oder `false` zurückliefert, erhalten konsumierende Anwendungen keine Berechtigungsstrukturen. Damit ist es nicht möglich, einen Policy Enforcement Point (PEP) zu implementieren, der Entscheidungen lokal oder in einem Intermediär trifft – jede Prüfung erfordert einen Roundtrip zum zentralen System.

**Problem 4 – Keine Partitionierbarkeit:**
Aktuell laufen alle Anfragen aller 1.500 DATEV-Anwendungen auf dasselbe System. Eine horizontale Partitionierung – z.B. nach Beraternummer oder Mandantenbereich – ist mit der aktuellen Architektur nicht möglich, da der Graph nicht segmentiert ist.

---

## 4. Warum dieses Risiko jetzt relevant wird

Das beschriebene Problem ist nicht neu – es ist ein bekanntes Skalierungsproblem zentraler Berechtigungssysteme. Es wird jedoch aus zwei Gründen jetzt akut:

**Cloud-Migration als Multiplikator:** Jede neue Cloud-Anwendung erhöht die Last auf das zentrale System. Mit 1.500 zu migrierenden Anwendungen ist das Wachstum der Anfragen nicht linear, sondern exponentiell – jede neue Anwendung bringt nicht nur eigene Anfragen, sondern multipliziert die Mandanten- und Mitarbeiterkombinationen.

**Data-Next als Frühindikator:** Datenplattformen wie Data-Next exponieren Daten über Output Ports, die von vielen Konsumenten gleichzeitig abgerufen werden. Jeder Datenzugriff erfordert eine Berechtigungsprüfung – die Last auf das Cloud-Rechtemanagement würde durch solche Plattformen überproportional steigen. Data-Next kann ohne eine skalierbare Berechtigungsarchitektur keine sinnvolle Tenant Isolation implementieren.

---

## 5. Lösungsrichtungen

Die folgenden Ansätze werden als Diskussionsgrundlage skizziert. Sie schließen sich nicht gegenseitig aus und können kombiniert werden.

### 5.1 Lokale Policy Decision (Dezentralisierung des PDP)

Anstatt jede Berechtigungsentscheidung zentral zu treffen, könnten konsumierende Anwendungen und Plattformen einen lokalen PDP betreiben. Dafür müsste das Cloud-Rechtemanagement nicht mehr `true/false` zurückliefern, sondern eine **Submenge der relevanten Berechtigungsregeln** für den aktuellen Kontext (Benutzer + Kanzlei + Anwendung).

Die konsumierende Anwendung führt die Policy Decision dann lokal aus – ohne Roundtrip zum zentralen System. Das zentrale System wird nur noch für die initiale Regelabfrage und bei Regeländerungen kontaktiert.

### 5.2 Content-Based Addressing für sicheres Caching

Um lokale Caches sicher invalidieren zu können, könnte die Rechtsmenge für einen gegebenen Kontext als **Hash** repräsentiert werden. Da ein Hash deterministisch von seinem Inhalt abhängt, ändert er sich automatisch bei jeder Änderung der zugrundeliegenden Berechtigungen – ohne dass ein expliziter Cache-Invalidierungsmechanismus benötigt wird.

Das Prinzip:
1. Konsumierende Anwendung holt beim Login den Hash der relevanten Rechtsmenge
2. Bei jedem Zugriff prüft sie lokal, ob der Hash noch aktuell ist
3. Nur bei Hash-Änderung wird die neue Rechtsmenge vom zentralen System geholt
4. Die Policy Decision wird lokal auf Basis der gecachten Rechtsmenge ausgeführt

Dies ermöglicht lange Cache-Laufzeiten ohne Sicherheitsrisiko – der Cache ist immer konsistent mit dem tatsächlichen Berechtigungsstand.

### 5.3 Kontext-Entität als Pflichtbestandteil des Tokens

Die DATEV-Konto-Identität allein reicht für eine Berechtigungsentscheidung nicht aus. Es braucht zusätzlich die **Kontext-Entität**: für welche Kanzlei agiert der Benutzer gerade, in welcher Rolle. Erst die Kombination aus Identität und Kontext-Entität erlaubt eine granulare, mandantenscharfe Policy Decision.

Diese Kontext-Entität sollte als Pflichtbestandteil des erweiterten Tokens definiert werden – nicht als optionaler Zusatz – damit PEPs auf dieser Grundlage grob-granulare Vorabprüfungen durchführen können, bevor eine feingranulare Entscheidung notwendig wird.

### 5.4 Partitionierung nach Ordnungsbegriffen

Da die Beraternummer als Partitionierungsschlüssel geeignet ist (jede Anfrage ist einer Kanzlei zuordenbar), könnte der Berechtigungsgraph nach Beraternummer partitioniert werden. Dies ermöglicht eine horizontale Skalierung des Cloud-Rechtemanagements – verschiedene Instanzen bedienen verschiedene Kanzleibereiche.

### 5.5 Technologiespezifische PEPs

Da Output Ports in einer Datenplattform unterschiedliche Technologien exponieren können (REST, Kafka, SQL, Object Storage, GraphDB), braucht jede Technologie einen eigenen Policy Enforcement Point. Die Berechtigungslogik selbst bleibt dabei generisch – nur der Enforcement-Mechanismus ist technologiespezifisch. Eine zentrale PEP-Bibliothek, die für verschiedene Technologien instanziiert werden kann, wäre effizienter als technologiespezifische Einzellösungen.

---

## 6. Handlungsbedarf

Das aktuelle Cloud-Rechtemanagement funktioniert für seinen heutigen Scope. Mit fortschreitender Cloud-Migration wird es jedoch zu einem systemischen Engpass, der das Ziel einer vollständigen Cloud-Transformation gefährdet.

Konkret empfohlen wird:

- **Kurzfristig:** Bewertung ob das Cloud-Rechtemanagement neben `true/false` auch Berechtigungssubmengen ausliefern kann – dies ist die Voraussetzung für alle weiterführenden Maßnahmen
- **Mittelfristig:** Definition eines standardisierten Kontext-Token-Formats (Identität + Kontext-Entität) als Grundlage für dezentrale Policy Decisions
- **Mittelfristig:** Konzept für Content-Based Caching der Berechtigungssubmengen
- **Langfristig:** Partitionierungskonzept für den Berechtigungsgraphen nach Ordnungsbegriffen

---

## 7. Bezug zu laufenden Architekturvorhaben

Dieses Risiko ist unmittelbar relevant für alle Cloud-nativen Vorhaben bei DATEV, die Berechtigungsprüfungen benötigen. Exemplarisch:

- **Data-Next Plattform:** Kann ohne skalierbare Berechtigungsarchitektur keine Tenant Isolation auf Output-Port-Ebene implementieren (siehe ADR-TI-001)
- **Neue Cloud-Anwendungen allgemein:** Jede Anwendung die das Cloud-Rechtemanagement als einzige Berechtigungsquelle nutzt, erbt das Skalierungsrisiko

---

*ARN-IAM-001 · DATEV Cloud-Architektur · Stand: 17.03.2026 · Status: DRAFT*
