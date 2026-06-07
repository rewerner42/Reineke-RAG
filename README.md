# Reineke-RAG

[![tests](https://github.com/rewerner42/Reineke-RAG/actions/workflows/tests.yml/badge.svg)](https://github.com/rewerner42/Reineke-RAG/actions/workflows/tests.yml)
![offline](https://img.shields.io/badge/Betrieb-100%25%20offline-2ea44f)
![python](https://img.shields.io/badge/Python-3.11%2B-3776ab)
![nutzung](https://img.shields.io/badge/Nutzung-intern-lightgrey)

Lokales, vollständig **offline lauffähiges RAG-System** (Retrieval-Augmented
Generation): Es beantwortet natürlichsprachige Fragen über interne Dokumente —
Word, PDF, Excel, PowerPoint, HTML und MediaWiki-Exporte — und belegt jede
Antwort mit Quellen. Entwickelt für die interne Nutzung bei Reineke-Technik:
keine Cloud, keine externen API-Aufrufe, alle Daten bleiben on-premise.

> **Status:** lauffähige Implementierung in [`rag-qdrant-local/`](rag-qdrant-local/).
> Die ursprüngliche Konzeptphase (Docker-Compose-Stack, ADRs, mehrsprachige
> Handbücher) wurde durch diese schlankere FastAPI-Lösung ersetzt.

## Highlights

- **100 % offline / on-premise** — LLM und Embeddings laufen lokal über
  [Ollama](https://ollama.com/), Vektoren in [Qdrant](https://qdrant.tech/).
  Kein Dokumentinhalt verlässt die Maschine.
- **Mandantenfähig** — strikte Isolation pro `(tenant, project)`; jede
  Vektorsuche erzwingt beide Filter, ein Aufruf ohne sie wird abgelehnt.
- **Breite Formatunterstützung** — PDF, DOCX, DOC, XLSX, XLS, PPTX, ODT, HTML
  sowie automatisch erkannte **MediaWiki-XML-Exporte**. Legacy-Formate werden
  über LibreOffice konvertiert.
- **Hohe Retrieval-Qualität** — Bi-Encoder-Embeddings plus ein standardmäßig
  aktiver Cross-Encoder-**Reranker** (`BAAI/bge-reranker-v2-m3`, lokal; ab
  größeren Sammlungen automatisch, global abschaltbar) für schärfere Treffer.
- **Konversations-Gedächtnis** — ein Query-Rewriter löst Folgefragen mit
  Pronomen („welche von beiden ist strenger?") vor der Suche auf.
- **Keine Antwort ohne Quellen** — bleibt die Trefferqualität unter der
  Schwelle, antwortet das System mit festem Fallback-Text statt zu halluzinieren.
- **OpenAI-kompatibel** — `/v1/chat/completions` und `/v1/models` binden
  [OpenWebUI](https://openwebui.com/) und andere Clients out of the box an.
- **Admin-UI** (Bootstrap + htmx) — Ingest-Assistent mit Dateityp-Filter,
  Live-Fortschritt, Job-Logs, Zeitplan und editierbarem System-Prompt.
- **Offline-Installer** — air-gapped Bundle (Docker Compose + systemd,
  Backup/Restore) für Linux-Server ohne Internet-Zugang.

## Architektur in einem Satz

[FastAPI](https://fastapi.tiangolo.com/) + [Ollama](https://ollama.com/) +
[Qdrant](https://qdrant.tech/), Multi-Tenant über SQLite-Metadaten,
Datei-Ingestion aus serverseitig gemounteten Verzeichnissen, standardmäßig
aktiver Cross-Encoder-Reranker, OpenAI-kompatibler `/chat`-Endpunkt, eingebautes Admin-UI.

![Architektur-Schema](docs/architecture-schema.svg)

## Schnellstart (Entwicklung)

Voraussetzungen: laufende Instanzen von **Ollama** (`http://localhost:11434`)
und **Qdrant** (`http://localhost:6333`), Python 3.11+, optional LibreOffice
für `.doc`/`.xls`/`.odt`.

```bash
git clone git@github.com:rewerner42/Reineke-RAG.git
cd Reineke-RAG/rag-qdrant-local

cp .env.example .env
# .env editieren — vor allem ALLOWED_BASE_PATHS!

cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Benötigte Modelle (einmalig ziehen):

```bash
ollama pull mxbai-embed-large   # Embeddings
ollama pull qwen2.5:14b         # Chat
ollama pull qwen2.5:7b          # Query-Rewriter (optional, empfohlen)
```

- **Admin-UI:** <http://localhost:8000/admin>
- **API-Docs:** <http://localhost:8000/docs>
- **Healthcheck:** `curl -s http://localhost:8000/health | jq`

Vollständige Anleitung inkl. Modell-Pulls, Smoke-Test und OpenWebUI-Anbindung:
[`rag-qdrant-local/README.md`](rag-qdrant-local/README.md).

## Produktiv-Deployment (offline)

Für die Installation auf einem **air-gapped Linux-Server** erzeugt
[`scripts/build-bundle.sh`](scripts/build-bundle.sh) ein in sich geschlossenes
Bundle (`dist/reineke-rag-installer-<version>.tar.gz`): Docker-Compose-Stack,
systemd-Service inkl. Backup-Timer, idempotentes `install.sh`,
Backup-/Restore-/Diagnose-Skripte sowie das vorgebaute Backend-Docker-Image —
kein Internet nötig.

```bash
# 1) Bundle bauen (auf einer Maschine mit Docker)
scripts/build-bundle.sh

# 2) Bundle auf den Zielserver kopieren, entpacken und installieren
sudo tar xzf reineke-rag-installer-<version>.tar.gz -C /tmp
cd /tmp/reineke-rag-installer-<version>
sudo ./install.sh            # idempotent; --dry-run prüft nur Voraussetzungen
```

Details, Hardware-Empfehlungen und air-gapped-Auslieferung (vorgepullte Modelle):
[`rag-qdrant-local/installer/README.md`](rag-qdrant-local/installer/README.md).

## Unterstützte Dateiformate

| Format | Verarbeitung |
| ------ | ------------ |
| `.pdf` | `pypdf`, seitenweise Text (ohne Textebene → `requires_ocr`) |
| `.docx`, `.doc` | `python-docx`; `.doc` via LibreOffice konvertiert; Tabellen → Markdown |
| `.xlsx`, `.xls` | `openpyxl`; `.xls` via LibreOffice; Chunks pro Zeilenblock, Header wiederholt |
| `.pptx` | Folientext und -tabellen |
| `.odt` | OpenDocument-Text; via LibreOffice → DOCX |
| `.html`, `.htm` | BeautifulSoup; Skripte/Styles/Navigation entfernt, Tabellen → Markdown |
| MediaWiki-XML | Export wird am `<mediawiki>`-Wurzelelement automatisch erkannt |

## Repo-Layout

| Pfad | Inhalt |
| ---- | ------ |
| [`rag-qdrant-local/`](rag-qdrant-local/) | Backend-Implementierung — **Start hier.** Eigene `README.md` mit ausführlichem Schnellstart. |
| [`rag-qdrant-local/backend/`](rag-qdrant-local/backend/) | FastAPI-App, Tests, `Dockerfile`, `requirements.txt` |
| [`rag-qdrant-local/backend/app/admin/`](rag-qdrant-local/backend/app/admin/) | Admin-UI (Bootstrap + htmx) |
| [`rag-qdrant-local/backend/app/connectors/`](rag-qdrant-local/backend/app/connectors/) | Quell-Connectoren (z. B. MediaWiki-Export) |
| [`rag-qdrant-local/installer/`](rag-qdrant-local/installer/) | Offline-Installer (Compose, systemd, Backup/Restore) |
| [`rag-qdrant-local/docs/`](rag-qdrant-local/docs/) | Backend-Doku: OpenWebUI-Pipe, MediaWiki-Connector, Ollama-Tuning |
| [`docs/TECHNISCHE_DOKUMENTATION.md`](docs/TECHNISCHE_DOKUMENTATION.md) | Technische Gesamtdokumentation (deutsch) |
| [`pdf/TECHNISCHE_DOKUMENTATION.pdf`](pdf/TECHNISCHE_DOKUMENTATION.pdf) | Gerendertes PDF derselben Doku |
| [`scripts/md2pdf.py`](scripts/md2pdf.py) | WeasyPrint-Renderer (Markdown → paginiertes PDF) |

## Sicherheit

- **Allow-list erlaubter Wurzelpfade** (`ALLOWED_BASE_PATHS`) — Nutzer können
  nur Dateien indexieren, die physisch darunter liegen.
- **Path-Traversal-Schutz** — Eingaben werden kanonisiert und gegen Allow-list
  und eine System-Deny-list (`/etc`, `/root`, …) geprüft.
- **Mandanten-Isolation** — jede Qdrant-Suche enthält zwingend `tenant`- und
  `project`-Filter.
- **Quellenzwang** — ohne ausreichend relevante Treffer wird das Chat-Modell
  gar nicht erst aufgerufen.

## Tests

```bash
cd rag-qdrant-local/backend
pytest
```

CI führt die Suite auf jedem PR aus (Python 3.11 + 3.12, GitHub Actions).
Zusätzlich gibt es ein versioniertes Retrieval-Quality-Eval
(Recall@5 / MRR / Faithfulness / Latenz) — siehe
[`rag-qdrant-local/README.md`](rag-qdrant-local/README.md#tests).

## Dokumentation rendern

```bash
python scripts/md2pdf.py \
  docs/TECHNISCHE_DOKUMENTATION.md \
  pdf/TECHNISCHE_DOKUMENTATION.pdf \
  --schema docs/architecture-schema.svg
```

## Lizenz

Internes Projekt der Reineke-Technik. Keine Veröffentlichungslizenz
festgelegt — Nutzung außerhalb des Unternehmens nur nach Absprache.
