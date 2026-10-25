# ML Lab — instruções para o Claude

Laboratório de machine learning feito para aprender, fase a fase. O plano completo está em
`docs/architecture.md`; as decisões e as suas razões, em `docs/adr/`.

## Como trabalhar

- Escreve em português de Portugal (respostas, documentação, docstrings, mensagens); os
  nomes no código ficam em inglês. Docstrings no estilo Google.
- O projeto é para aprender: explica o porquê das escolhas, não só o quê.
- Trabalha fase a fase, pelo roadmap de `docs/architecture.md`. Uma fase só termina
  quando o critério "Pronto quando" se cumpre, `ruff`, `mypy` e `pytest` passam e as
  fichas didáticas dos algoritmos novos estão em `docs/algorithms/`.
- Cada decisão que não seja óbvia fica numa ADR em `docs/adr/` (o modelo está no
  `README.md` dessa pasta) e nas notas da fase em `docs/architecture.md`.
- Commits só quando pedidos, em ramos `feat/...` ou `fix/...`, no formato Conventional
  Commits.

## Comandos

```bash
uv sync                                          # instalar
uv run ruff format .                             # formatar
uv run ruff check .                              # lint
uv run mypy                                      # tipos (strict)
uv run pytest                                    # todos os testes
uv run pytest -m "not integration and not api"   # ciclo rápido
uv run streamlit run ui/simple/app.py            # a interface simples
uv run python -m ml_lab.api                      # a API (documentação em /docs)
```

Os avisos contam como erros nos testes: resolve-os em vez de os silenciar.

## Arquitetura (as regras têm testes em `tests/architecture/`)

- `core/` nunca importa `api/` nem `ui/`; `core/foundation/` não importa nada do projeto;
  o pandas só existe em `core/data/`.
- A interface e a API só importam `ml_lab.core.services` e `ml_lab.core.schemas` (e
  `config`, `logging`, `formatting`, `process`).
- Os componentes registam-se (`MODELS`, `METRICS`, `EVALUATORS`, `GENERATORS`, `AGENTS`,
  `ENVIRONMENTS`…). Um algoritmo novo é um ficheiro novo mais a sua ficha, e os testes de
  contrato testam-no sozinho. Ver "Manutenção: como adicionar componentes" em
  `docs/architecture.md`.
- O PyTorch só existe nas redes (`core/models/neural/`, `torch_adapter.py` e o DQN em
  `core/rl/agents/dqn.py`); o SQLite e o joblib só em `core/tracking/`.

## O backend tem de ficar pronto para o frontend

A interface simples (Streamlit) é provisória. Na fase 10 é refeita em React, que fala com
o núcleo através de uma API HTTP (fase 7, FastAPI). **Todo o código do backend escreve-se
a pensar nisso**, para que a interface nova só tenha de desenhar (ADR 0019):

1. **Os serviços são a única porta.** Tudo o que um ecrã precisa existe como método
   público de um serviço em `core/services/`. Se a interface precisa de algo novo, cria-se
   no serviço, não na página.
2. **Os serviços devolvem dados simples.** Dataclasses `frozen` ou modelos Pydantic com
   texto, números, booleanos, `None`, enums, tuplos e dicionários. Nunca DataFrames,
   matrizes do NumPy, modelos treinados ou objetos com estado. Os números vão em bruto,
   com o que é preciso para os mostrar (por exemplo, `as_percent`).
3. **Tudo passa a JSON.** `tests/contract/test_service_boundary.py` exige que cada
   método público devolva um tipo que o Pydantic sabe descrever, e resultados reais que
   passam a JSON estrito. Para serializar, usa `ml_lab.core.schemas.serialization.to_json`
   (os NaN passam a `null`). Um método que só serve outros serviços (devolve DataFrames
   ou o modelo) leva `@internal`.
4. **As regras de domínio ficam no backend.** A interface não decide nada que o frontend
   também teria de decidir: que divisões existem para uma tarefa, o que significa o estado
   de um modelo, qual é a classe anómala. Pede-o a um serviço (por exemplo
   `DatasetService.task_options` ou `ExperimentResult.learned`). A interface decide só a
   apresentação: a disposição, os separadores e as cores.
5. **Os textos didáticos vêm do backend:** fichas, explicações das métricas, temas do
   painel "Porquê?" e avisos (`Notice`, com um código estável). Assim só se escrevem uma
   vez. Os textos novos vão para o backend. As legendas de gráficos que ainda estão nas
   páginas do Streamlit mudam-se quando os ecrãs forem refeitos.
6. **O trabalho longo corre em segundo plano**, com um estado serializável
   (`TrainingJob.state()` devolve um `JobState`). O serviço que o lança guarda-o num
   `JobRegistry` e expõe `job`, `pause`, `resume`, `cancel`, `updates` e `result` pelo id
   do run: é isso que a API devolve a quem acompanha o treino (ADR 0022).
7. **Identificadores estáveis, em texto:** o id do dataset, o id do run, o nome do
   modelo no registo, o código de cada aviso. É com eles que o frontend volta a pedir as
   coisas.
8. **Os erros são subclasses de `MLLabError`**, com uma mensagem em português que diz o
   que fazer. A API traduz a categoria num código HTTP, por isso escolhe a subclasse certa:
   `TrainingStateError` → 409, `DataError` e `ValueError` → 422, `LookupError` → 404,
   `TrackingError` → 500, outro `MLLabError` → 400 (`api/errors.py`).
9. **Cada método público de um serviço tem a sua rota** em `api/routes/<serviço>.py`, com
   o nome do método (`tests/api/test_contract.py` falha enquanto faltar).
