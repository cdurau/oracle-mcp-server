# Oracle MCP für DWH-Analysen

Der Server liefert Oracle-Metadaten und führt SQL über MCP aus. Typische Aufgaben sind Ladeprozesse nachvollziehen, PL/SQL-Packages lesen, Abhängigkeiten prüfen und langsame Abfragen untersuchen. Beispiele für MCP-Clients stehen im [README](README.md).

## Einrichtung

1. Python 3.12 oder neuer und `uv` installieren. Im Projektverzeichnis `uv sync --extra dev` ausführen.
2. Die Verbindung in der Umgebung des MCP-Prozesses oder in einer lokalen `.env` setzen. [.env.example](.env.example) zeigt das Verbindungsformat. Zugangsdaten aus Git und Logs heraushalten.
3. Den MCP-Client mit `uv run main.py` aus dem Projektverzeichnis starten. Der Server kommuniziert über stdio.

| Variable | Verhalten |
|---|---|
| `ORACLE_CONNECTION_STRING` | Standardvariable für den Oracle-Verbindungsstring. |
| `ORACLE_CONNECTION_STRING_VAR` | Optionaler Name einer anderen Variable für den Verbindungsstring, z. B. `ORACLE_CONNECTION_STRING_DWH_TEST`. Ein leerer Wert verwendet den Standard. Eine ausgewählte, aber fehlende Variable ergibt beim Start einen Fehler mit ihrem Namen. |
| `TARGET_SCHEMA` | Optionales Schema für Metadaten. Für frei formuliertes SQL bleibt die Namensauflösung der Oracle-Session maßgeblich. |
| `CACHE_DIR` | Verzeichnis für den Schema-Cache. Standard ist `.cache`. Pro Instanz mit anderem Ziel einen eigenen Pfad verwenden. |
| `READ_ONLY_MODE` | Standardmäßig aktiv. Schreibzugriff lässt sich mit `0`, `false` oder `no` freigeben und braucht zusätzlich die Oracle-Rechte. |
| `THICK_MODE` | Standard ist Thin. `1`, `true` oder `yes` fordert Thick mit Oracle Client an. Bei einem Fehler bei der Client-Initialisierung fällt der Code auf Thin zurück. |
| `ORACLE_CLIENT_LIB_DIR` | Optionaler Pfad zu den Oracle Client Libraries für Thick. |

## Funktionen und Arbeitsweise

- **Schema erkunden.** Mit `search_tables_schema` oder `search_columns` anfangen und Treffer mit `get_table_schema`, `get_table_constraints`, `get_table_indexes` und `get_related_tables` eingrenzen. `get_tables_schema` eignet sich für eine kurze Liste bekannter Tabellen. Den Cache nur bei Bedarf mit `rebuild_schema_cache` erneuern.
- **PL/SQL nachvollziehen.** `get_pl_sql_objects` listet Objekte. `get_object_source` liest unter anderem Package-Spezifikation und Package-Body. `get_dependent_objects` zeigt abhängige Objekte. `get_user_defined_types` und `get_database_vendor_info` liefern ergänzende Metadaten.
- **SQL und Pläne untersuchen.** `run_sql_query` startet standardmäßig mit maximal 100 Ergebniszeilen. Die Grenze beschränkt die Ausgabe, nicht die Arbeit in Oracle. `explain_query_plan` nutzt `PLAN_TABLE` und räumt mit `DELETE` auf. Für strikt lesende Prüfungen einen bereits vorhandenen Laufzeitplan auswerten, sofern die Rechte dafür vorhanden sind.
- **Produktion prüfen.** Zielverbindung vor der Abfrage kontrollieren und für reine Analysen den Lesemodus sowie ein Oracle-Konto mit Leserechten verwenden. Ladeprozesse und Package-Änderungen zuerst in der Testumgebung prüfen.

## Verbindungen und Session-Anzeige

- Der Pool verwendet 2 bis 10 Verbindungen pro Instanz und wartet, wenn keine Verbindung frei ist. Thin verwendet den asynchronen Pool und braucht keinen Oracle Client.
- Der Connector übergibt `PROGRAM = plsqldev.exe` beim Poolaufbau. Beim Ausleihen setzt er `MODULE = PL/SQL Developer` und `ACTION = SQL-Fenster`. Ungenutzte Poolverbindungen können deshalb andere Standardwerte anzeigen.
- Thin setzt `MACHINE` aus `USERDOMAIN` und dem System-Hostnamen zusammen, wenn eine vom Hostnamen verschiedene Domäne vorhanden ist. Sonst greift der Treiberstandard. Thick verwendet den Oracle Client für den Hostnamen. `OSUSER` wird vom Projekt nicht überschrieben.
- Nach Änderungen am Pool den MCP neu starten. Eine lesende Abfrage auf die eigene Session in `V$SESSION` prüft die angezeigten Werte sowie `SID` und `SERIAL#`. Die Anzeigewerte allein unterscheiden den MCP nicht zuverlässig vom Desktop-Client.

## Code und Tests

- [main.py](main.py) definiert Konfiguration, FastMCP-Tools und Startverhalten. [db_context/database.py](db_context/database.py) verwaltet Pool und SQL-Ausführung. [db_context/schema/](db_context/schema/) enthält Cache und Formatierung.
- Unit-Tests laufen ohne Datenbank mit `uv run --extra dev python -m pytest tests/unit`. Integrationstests stehen in [tests/integration/](tests/integration/) und brauchen eine dedizierte Oracle-Testverbindung in `ORACLE_CONNECTION_STRING`. Einige Tests schreiben Daten oder legen Objekte an.
