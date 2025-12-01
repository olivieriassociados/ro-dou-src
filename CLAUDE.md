# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ro-DOU is an Apache Airflow-based system that performs automated clipping (monitoring) of Brazil's Diário Oficial da União (DOU - Federal Official Gazette) and municipal gazettes via Querido Diário. It searches for user-defined keywords and sends notifications via email, Slack, or Discord.

## Development Commands

```bash
# Start the development environment (Docker + Airflow)
make run

# Stop all containers
make down

# Run tests (inside container)
make tests

# Alternative: Run tests directly in container
docker exec airflow-webserver sh -c "cd /opt/airflow/tests/ && pytest -vvv --color=yes"

# Run a single test file
docker exec airflow-webserver sh -c "cd /opt/airflow/tests/ && pytest -vvv --color=yes <test_file>.py"

# Run a specific test
docker exec airflow-webserver sh -c "cd /opt/airflow/tests/ && pytest -vvv --color=yes <test_file>.py::<test_function>"
```

## Architecture

### Core Components

- **`src/dou_dag_generator.py`**: Main DAG generator that reads YAML configs from `dag_confs/` and dynamically creates Airflow DAGs. The `DouDigestDagGenerator` class orchestrates searches and notifications.

- **`src/searchers.py`**: Three search implementations:
  - `DOUSearcher`: Searches via oficial DOU API
  - `QDSearcher`: Searches via Querido Diário API (municipal gazettes)
  - `INLABSSearcher`: Searches via INLABS PostgreSQL database (alternative DOU source)

- **`src/schemas.py`**: Pydantic models for YAML validation (`DAGConfig`, `SearchConfig`, `ReportConfig`, etc.)

- **`src/parsers.py`**: `YAMLParser` class that loads and validates YAML configuration files

- **`src/hooks/`**: Airflow hooks for data sources
  - `dou_hook.py`: DOU API hook
  - `inlabs_hook.py`: PostgreSQL hook for INLABS database with SQL generation

- **`src/notification/`**: Notification senders implementing `ISender` interface
  - `email_sender.py`, `slack_sender.py`, `discord_sender.py`
  - `notifier.py`: Orchestrates multiple notification channels

### DAG Configuration

DAGs are defined via YAML files in `dag_confs/`. Each YAML generates an Airflow DAG that:
1. Fetches search terms (from list, Airflow variable, or database)
2. Executes searches across configured sources (DOU, QD, INLABS)
3. Sends notifications with results

Example YAML structure:
```yaml
dag:
  id: my_dag_name
  description: Description
  search:
    terms:
      - keyword1
      - keyword2
    sources: [DOU]  # or [QD], [INLABS], [DOU, QD]
    dou_sections: [TODOS]  # SECAO_1, SECAO_2, SECAO_3, etc.
  report:
    emails:
      - user@example.com
    subject: "Report Subject"
```

### Environment Variables

- `RO_DOU__DAG_CONF_DIR`: Path to YAML config directory (required)
- `AIRFLOW__CORE__DEFAULT_TIMEZONE`: Timezone for scheduling

### Docker Setup

The project runs on Docker with:
- PostgreSQL database (Airflow metadata + INLABS data)
- Airflow webserver (port 8080)
- Airflow scheduler
- smtp4dev for email testing (port 5001 for web UI)

Volume mounts map `src/` to `/opt/airflow/dags/ro_dou_src` and `dag_confs/` to `/opt/airflow/dags/ro_dou/dag_confs`.

## Testing

Tests are located in `tests/` and use pytest. Key fixtures in `conftest.py`:
- `dag_gen`: `DouDigestDagGenerator` instance
- `yaml_parser`: Parser for basic_example.yaml
- `dou_searcher`, `inlabs_searcher`: Searcher instances
- `report_example`, `search_results`: Sample data for testing

Tests run inside the Airflow container to have access to Airflow dependencies.

## Key Patterns

- Search results use a nested dict structure: `{group: {term: {department: [results]}}}`
- The `merge_results()` function combines results from multiple search sources
- Pydantic models in `schemas.py` validate all YAML configuration
- Notification senders implement the `ISender` protocol with `send_report()` method
