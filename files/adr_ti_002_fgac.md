# ADR-TI-002: Fine-Grained Access Control (FGAC)

| | |
|---|---|
| **Status** | DRAFT |
| **Version** | 1.0 |
| **Datum** | 09.03.2026 |
| **Autor** | Marcel Meyer / adorsys GmbH & Co. KG |
| **Kontext** | DATEV Data-Next Plattform – Handlungsfeld Datenplattform |
| **Verwandte ADRs** | ADR-TI-001 (Tenant-Modell), ADR-TI-003 (Datenprodukt-Governance) |

---

## 1. Problemstellung

Nachdem ADR-TI-001 klärt, wer ein Tenant ist und wie Identität aufgelöst wird, adressiert dieses ADR die nächste Ebene: Wer darf auf welche Daten zugreifen – und wie wird das durchgesetzt?

Die Data-Next Plattform exponiert Daten über Output Ports in unterschiedlichen Technologien (SQL, REST, Kafka, Object Storage, GraphDB). Jede Technologie hat ein eigenes Zugriffsmodell. Eine einheitliche, technologieneutrale Berechtigungsarchitektur existiert nicht. Gleichzeitig sind die Berechtigungslogiken im DATEV-Kontext feingranular, kontextabhängig und dynamisch – statische Zugriffsregeln reichen nicht aus.

---

## 2. Zweistufiges Berechtigungsmodell

Der Zugriff auf ein Datenprodukt unterliegt zwei konzeptionell getrennten Ebenen:

**Ebene 1 – Subscription (Anwendung → Datenprodukt):**
Die Anwendung ist als Konsument berechtigt, den Output Port grundsätzlich anzusprechen. Dies wird über den Data Contract in Witboost geregelt. Diese Ebene ist M2M – kein Benutzerkontext ist involviert.

**Ebene 2 – FGAC (Benutzer → Daten):**
Innerhalb des Datenprodukts muss sichergestellt werden, dass ein Benutzer nur die Daten sieht, auf die er im DATEV-Berechtigungskontext berechtigt ist. Diese Ebene ist kontextabhängig und muss je Request ausgewertet werden.

Diese beiden Ebenen dürfen nicht vermischt werden. Eine Lösung die nur Ebene 1 adressiert, löst das FGAC-Problem nicht. Eine Lösung die nur Ebene 2 adressiert, lässt die Subscription-Kontrolle außer Acht.

---

## 3. Feingranulare Berechtigungen im DATEV-Kontext

Die Berechtigungslogik unterscheidet sich je nach Datenprodukt erheblich. Zwei Beispiele aus dem PWS-Kontext:

**Arbeitnehmerstammdaten:**
- Ein Steuerberater (Inhaber) sieht alle Arbeitnehmerstammdaten seiner Kanzlei
- Ein Mitarbeiter darf ggf. nur seine eigenen Stammdaten sehen
- Berechtigungen sind an die Rolle innerhalb der Kanzlei gekoppelt

**Mandantenstammdaten:**
- Ein Steuerberater (Inhaber) sieht alle Mandanten der Kanzlei
- Mitarbeiter sehen nur die Mandanten, die ihnen zugewiesen sind
- Die Zuweisung ist eine kanzleiinterne Entscheidung – nicht durch die Plattform steuerbar

Zusätzlich existieren **delegierte Rechte über Vollmachten**: Interne DATEV-Abteilungen können im Auftrag einer Kanzlei auf Daten zugreifen – diese Berechtigung hängt von der expliziten Zustimmung der Kanzlei ab, die dynamisch erteilt und entzogen werden kann.

---

## 4. Technologievarianten und ihre FGAC-Implikationen

Der Begriff „Row-Level Access" entstammt der relationalen Datenbankwelt und greift für die Data-Next Plattform zu kurz. Jede Output-Port-Technologie hat ein anderes Zugriffsmodell:

| Technologie | Zugriffsmodell | Komplexität der FGAC |
|---|---|---|
| **Relationale DB / SQL** | Row-Level Security, Column Masking | Gut etabliert (z.B. OPA/Trino, PostgreSQL RLS) |
| **REST API** | Kein natürliches Row-Konzept – Filtering muss in API-Logik oder Gateway implementiert werden | Mittel – abhängig von API-Design |
| **Object Storage (S3/Blob)** | Zugriff auf Objekt- oder Prefix-Ebene – kein Filtering innerhalb eines Objekts | Gering granular – Isolation nur auf Datei-/Bucket-Ebene möglich |
| **NoSQL (Document/Key-Value)** | Zugriff auf Dokument-Ebene – Filtering je nach Store unterschiedlich | Variabel – stark technologieabhängig |
| **Graph DB (z.B. Metaphactory)** | Graph-Traversal kann beliebig viele Knoten und Kanten berühren – Filtering auf Teilgraphen ist nicht trivial | Hoch – erfordert spezifische Graph-Berechtigungsmodelle |
| **Kafka** | Zugriff auf Topic-Ebene – feingranulares Filtering auf Nachrichtenebene erfordert eigene Interceptoren | Hoch – kein nativer FGAC-Mechanismus |

Übergreifende Konzepte:
- **Fine-Grained Access Control (FGAC):** Attributbasierte, kontextabhängige Zugriffssteuerung unterhalb der Objekt-Ebene
- **Context-Scoped Access Control:** Zugriff wird durch den Kontext des Aufrufers (Kanzlei, Rolle, Mandant) bestimmt
- **Tenant-Scoped Access Control:** Isolation auf Tenant-Ebene als primäres Steuerungskriterium

---

## 5. Prozessuale vs. technische Durchsetzung

Eine grundlegende Meta-Frage ist, ob Berechtigungsmanagement technisch enforced werden muss oder ob prozessuale Kontrollen ausreichen:

**Technisches Enforcement:** Berechtigungsentscheidungen werden zur Laufzeit von der Plattform durchgesetzt – unabhängig vom Verhalten der konsumierenden Anwendung. Erfordert einen Policy Enforcement Point (PEP) vor dem Output Port.

**Prozessuales Enforcement:** Konsumierende Anwendungen verpflichten sich über den Data Contract zur korrekten Handhabung von Benutzerkontexten. Kontrolle erfolgt durch Audits und vertragliche Vereinbarungen.

Die Antwort auf diese Frage ist nicht einheitlich – sie hängt vom Schutzbedarf der Daten ab. Für personenbezogene Daten (Arbeitnehmerstammdaten, Mandantenstammdaten) ist technisches Enforcement kaum vermeidbar. Für aggregierte Analysedaten könnte prozessuales Enforcement ausreichen.

---

## 6. Auditierbarkeit

Unabhängig vom gewählten Enforcement-Ansatz müssen Datenzugriffe auditierbar sein:

- Steuerrechtliche Vorschriften und DSGVO verlangen in bestimmten Kontexten die Nachweisfähigkeit von Datenzugriffen
- Bei einem Sicherheitsvorfall muss rekonstruierbar sein, welche Daten über welchen Kanal abgeflossen sind
- Wenn eine Anwendung den UserContext übermittelt und der Output Port darauf vertraut, muss dieser Vertrauensakt protokollierbar sein

Ungeklärt ist, auf welcher Ebene Audit-Logs erzeugt werden – am Output Port, in einem zentralen Gateway, oder in der konsumierenden Anwendung.

---

## 7. Offene Entscheidungsfragen

| # | Entscheidungsfrage | Verantwortlich |
|---|---|---|
| 1 | Muss FGAC technisch enforced werden, oder kann es für bestimmte Datenprodukt-Typen prozessual gelöst werden? Gilt die Antwort einheitlich oder schutzstufen-abhängig? | Data Governance + Legal + EAM |
| 2 | Wer ist verantwortlich für die Definition der FGAC-Regeln: Data Product Owner, konsumierende Anwendung oder Plattform? | Data Governance + EAM |
| 3 | An welchem Punkt in der Architektur wird FGAC durchgesetzt – am Output Port, in einem Intermediär oder in der konsumierenden Anwendung? *(abhängig von Frage 2)* | Solution Architect + Platform Team |
| 4 | Kann FGAC plattformweit einheitlich durchgesetzt werden, oder muss jede Output-Port-Technologie einen eigenen Enforcement-Mechanismus implementieren? | Solution Architect + Platform Team |
| 5 | Wie werden delegierte Rechte (Vollmachten) im Datenzugriffskontext technisch abgebildet? | Data Governance + Legal + IAM |
| 6 | Auf welcher Ebene werden Audit-Logs für Datenzugriffe erzeugt, wer ist für deren Aufbewahrung zuständig, und welche Compliance-Anforderungen (DSGVO, Steuerrecht) müssen erfüllt werden? | Solution Architect + Legal + Platform Team |

---

## 8. Randbedingungen für den Lösungsraum

**RB-TI-002-F1 – Subscription und FGAC müssen getrennt behandelt werden**
Eine Lösung muss Ebene 1 (Subscription, M2M) und Ebene 2 (FGAC, Benutzerkontext) konzeptionell und technisch trennen. Ein Mechanismus der beides vermischt ist nicht akzeptabel.

**RB-TI-002-F2 – Technologieneutralität**
Eine Lösung darf nicht implizit SQL/Row-Level-Security voraussetzen. Sie muss für alle relevanten Output-Port-Technologien (REST, Kafka, SQL, Object Storage, GraphDB) anwendbar sein oder explizit definieren, welche Technologien einen eigenen Enforcement-Mechanismus benötigen.

**RB-TI-002-F3 – Kontextabhängige Auswertung**
FGAC-Regeln müssen zur Laufzeit kontextabhängig auswertbar sein. Statische, benutzerunabhängige Zugriffsregeln sind für den DATEV-Kontext nicht ausreichend.

**RB-TI-002-F4 – Vollmachten müssen abbildbar sein**
Eine Lösung muss delegierte Rechte (Vollmachten) abbilden können. Binäre Berechtigungsmodelle ohne Delegationskonzept sind unzureichend.

**RB-TI-002-F5 – Auditierbarkeit ist Pflicht**
Jede Lösung muss Datenzugriffe auditierbar machen. Es muss nachvollziehbar sein, wer wann mit welchem Kontext auf welche Daten zugegriffen hat.

**RB-TI-002-S1 – Schutzstufen-Differenzierung**
Eine Lösung sollte zwischen Datenprodukt-Typen und Schutzstufen differenzieren können. Nicht alle Datenprodukte erfordern dasselbe Enforcement-Niveau.

---

## 9. Abgrenzung

Dieses ADR behandelt ausschließlich die Fragen der FGAC-Logik, Berechtigungsebenen und Enforcement-Mechanismen. Fragen zur Tenant-Definition und Identitätsstruktur sind Gegenstand von ADR-TI-001. Fragen zur Ownership von Berechtigungsregeln im Data-Mesh-Kontext sind Gegenstand von ADR-TI-003.

---

*ADR-TI-002 · DATEV Data-Next Plattform · Stand: 09.03.2026 · Status: DRAFT · Version 1.0*
