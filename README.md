# Projeto Aplicado — Data Warehouse e Data Lake

Este repositório inicia o projeto final com o dataset **Barcelona Luxury Hotel F&B**. Ele foi escolhido por conter várias tabelas fato e dimensão, histórico diário e métricas operacionais suficientes para comparar ETL em Data Warehouse com ELT em um Lakehouse Medallion.

## Arquiteturas entregues

| Camada | Implementação proposta | Evidência no projeto |
| --- | --- | --- |
| Data Warehouse (ETL) | PostgreSQL, esquema estrela | `sql/dw/` |
| Data Lake (ELT) | PySpark + Delta Lake, Medallion | `src/lakehouse/medallion.py` |
| Qualidade | Validação do dado fonte e regras de rejeição | `scripts/validate_sources.ps1` |
| Consultas analíticas | KPIs de receita, covers e orçamento | `sql/analytics/01_kpis.sql` |

## Dataset escolhido

- Fonte principal: `data/raw/barcelona/`
- Fonte alternativa, não usada como base do projeto: `data/raw/credit_card/`
- Justificativa e limitações: `docs/data-assessment.md`

Os dados são sintéticos. O arquivo `data/reference/unit_scd2_changes.csv` registra duas mudanças de negócio controladas, necessárias para demonstrar SCD Tipo 2, pois a fonte original traz apenas o retrato atual das unidades.

## Como executar

1. No VS Code, execute a tarefa **Validar fontes Barcelona** (`Terminal > Run Task`). Ela usa apenas PowerShell e funciona neste computador.
2. Para o Data Warehouse, crie um banco PostgreSQL e execute:
   - `sql/dw/00_create_schemas.sql` e `sql/dw/01_create_star_schema.sql`;
   - execute `sql/dw/03_etl_load.sql` uma primeira vez para criar as tabelas `stg.raw_*`;
   - importe os CSVs e `data/reference/unit_scd2_changes.csv` para as respectivas tabelas de estágio com `\copy`;
   - execute novamente `sql/dw/03_etl_load.sql` para preencher a área SCD, depois `sql/dw/02_scd2_unit.sql` e uma última vez `sql/dw/03_etl_load.sql` para carregar todos os fatos;
   - `sql/analytics/01_kpis.sql` para as análises.
3. Para o Lakehouse, instale Python 3.11+, depois execute:

   ```powershell
   py -m venv .venv
   .\.venv\Scripts\Activate.ps1
   pip install -r requirements.txt
   python src\lakehouse\medallion.py --raw-root data\raw\barcelona --lake-root data\lakehouse
   ```

O pipeline Delta grava as zonas `bronze`, `silver` e `gold`. Reexecutá-lo com uma nova versão da fonte demonstra time travel; adicionar uma coluna em um CSV de entrada demonstra schema evolution via `mergeSchema`.

No Windows, após a configuração inicial, a forma recomendada de executar o Lakehouse é:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\run_lakehouse.ps1 -Reset
```

Os logs de execução e as evidências ficam em `logs/`. O lote `data/schema-evolution/fact_revenue.csv` é usado para demonstrar evolução de schema sem alterar a fonte original.

Para gerar o dashboard HTML local após a carga do DW:

```powershell
$env:PGPASSWORD = '<senha do PostgreSQL local>'
python scripts\generate_dashboard.py
```

O resultado é salvo em `artifacts/dashboard.html` e pode ser aberto diretamente no navegador.

## Critérios avançados

- **SCD Tipo 2:** `sql/dw/02_scd2_unit.sql` e a dimensão `gold.dim_unit_scd2`.
- **Índices e particionamento:** fatos particionados por `date_key` e índices para os joins analíticos no DW.
- **Time travel:** consulta Delta ilustrada no fim de `medallion.py`.
- **Schema evolution:** escrita Bronze com `mergeSchema=true`.


## Entrega final e evidencias

Al?m dos scripts e dados, o pacote final inclui evid?ncias separadas para confer?ncia r?pida:

- `dw_projeto.duckdb`: snapshot port?til do DW para inspe??o local das dimens?es, fatos, SCD2, ?ndices e contagens.
- `pipeline_log.json`: resumo estruturado das etapas, status, evid?ncias e contagens antes/depois.
- `docs/evidencias/`: arquivos separados para SCD2, ?ndices, particionamento, time travel, schema evolution e valida??o de contagens.
- `artifacts/apresentacao_final.pptx`: apresenta??o exportada em PowerPoint.
- `artifacts/dashboard.html`: dashboard offline.

O DW oficial est? descrito nos scripts PostgreSQL em `sql/dw/`. O arquivo DuckDB ? uma c?pia de inspe??o para facilitar a corre??o sem depender de um servidor PostgreSQL local.

Para recriar o snapshot DuckDB, instale `duckdb` no Python usado e execute:

```powershell
python scripts\generate_duckdb_snapshot.py
```

Para capturar evid?ncias de EXPLAIN no PostgreSQL, use:

```powershell
psql -d barcelona_dw -f sql/analytics/04_explain_indexes.sql
```

## Estrutura

```text
data/raw/              # fontes extraídas dos ZIPs fornecidos
data/reference/        # eventos de mudança controlados para SCD2
docs/                  # decisão de dataset e arquitetura
scripts/               # validações reproduzíveis
sql/dw/                # modelo e carga do Data Warehouse
sql/analytics/         # consultas de negócio
src/lakehouse/         # pipeline Medallion Delta Lake
```
