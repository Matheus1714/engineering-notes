# ITA API – Proposta Inicial

> **Contexto:** API para consulta centralizada às provas de vestibular e de entrada da pós-graduação do ITA.

## Summary

Proposta de uma API de **somente leitura** para provas do vestibular e da pós-graduação do ITA. O resultado esperado é um ponto único de acesso para listar provas, obter detalhes e questões (com gabarito opcional), a partir de dados extraídos do site oficial e de arquivos locais, transformados em JSON padronizado e carregados em banco estruturado.

*What real problem does this solve?* Hoje não existe um repositório centralizado e fácil de consumir para essas provas; o acesso depende de um sistema legado e de fontes dispersas.

## Motivation / Problem Statement

Não há hoje um local que centralize as provas do ITA (vestibular e pós-graduação) em formato padronizado e acessível. As provas estão em um sistema legado de difícil acesso, o que atrapalha estudos, pesquisas e ferramentas que dependem desses dados.

*Why now?* A demanda por dados estruturados para estudo e para ferramentas (simulados, análise por tópico, etc.) torna relevante definir agora o modelo de dados e o pipeline (ETL + API) para que evoluções (ex.: associação a tópicos, simulados com IA) possam ser feitas em cima de uma base estável.

## Goals / Non-Goals

**Goals:**

- **Fase 1**
  - ETL: extração (web + arquivos locais) → transformação em JSON estruturado → carga em banco.
  - API de consulta: listar provas, obter detalhe da prova e questões com gabarito opcional.

- **Fase 2**
  - Associação de questões com [tópicos do programa](https://www.vestibular.ita.br/programa_2026.pdf).
  - Geração aleatória de prova com questões.
  - Análise de candidatos por região.

- **Fase 3**
  - Geração de simulado com IA.

**Non-Goals:**

- Não substituir o site oficial do vestibular; a API é complementar e para consumo programático.
- Não incluir, nesta proposta, autenticação de usuário ou controle de acesso (pode ser tratado depois).
- Não cobrir provas de outras instituições; escopo limitado ao ITA (vestibular e pós).

## Proposed Solution

### Visão geral do pipeline de dados (ETL)

O primeiro desafio é o ETL: obter os dados (site oficial e/ou arquivos locais), normalizá-los em um formato único e carregá-los em um banco para servir a API.

Fonte principal de referência: área de provas em https://www.vestibular.ita.br/.

![Fluxo do ETL – Extração, Transformação e Carga](../../.github/imgs/0001-etl.png)  
_Figura 1: ETL das provas (Tests)._

**Explicação da figura:**

1. **Extraction (Extração)**  
   Dados são obtidos por **scraping do site** e/ou a partir de **arquivos locais** já disponíveis. O resultado são arquivos PDFs que representam as provas em formato bruto.

2. **Transform (Transformação)**  
   Esses arquivos são processados e convertidos em **JSON estruturado**, com campos como `title`, `year` e `questions` (lista de questões), permitindo um modelo único independente do formato de origem.

3. **Load (Carga)**  
   O JSON gerado é **carregado em um banco de dados estruturado**, onde a API irá consultar para listar provas, detalhes e questões.

Com esse pipeline, a API atua apenas na camada de leitura sobre o banco já populado.

### Estrutura das provas (para desenho do modelo e ETL)

Para desenhar o modelo de dados e os transformadores, é necessário considerar a estrutura real das provas:

**Vestibular ITA**

| Período   | Conteúdo |
|----------|----------|
| 2008–2018 | Física: 30 (20 objetivas + 10 discursivas). Português: 20 objetivas + redação. Inglês: 20 objetivas. Matemática: 30 (20 objetivas + 10 discursivas). Química: 30 (20 objetivas + 10 discursivas). |
| 2019–2024 | **1ª fase:** 15 Física, 15 Português, 10 Inglês, 15 Matemática, 15 Química (objetivas). **2ª fase:** Matemática, Física e Química com 10 discursivas cada + Redação. |
| 2025–atual | **1ª fase:** 12 Matemática, 12 Física, 12 Química, 12 Inglês. **2ª fase:** 10 discursivas (Matemática, Física, Química), Português 15 objetivas + Redação. |

**Pós-graduação ITA**

- GMAT: 15 ou 16 questões (inglês ou português).
- Inglês: redação ou múltipla escolha.

### Contratos da API (resumo)

Enums e schemas de referência (exemplo em Python):

```python
class Discipline(Enum):
  FISICA = 'fisica'
  MATEMATICA = 'matematica'
  QUIMICA = 'quimica'
  PORTUGUES = 'portugues'
  INGLES = 'ingles'

class Language(Enum):
  PORTUGUES = 'portugues'
  INGLES = 'ingles'

class TestType(Enum):
  VESTIBULAR = 'vestibular'
  POS = 'pos'

class FormatType(Enum):
  ESSAY = 'essay'
  OBJECTIVE = 'objective'
```

```python
from typing import List, Optional

class TestSummary(Schema):
  id: str
  type: TestType
  year: int
  name: str
  number_of_questions: int
  language: Language
  format: List[FormatType]

ListTestsOut = List[TestSummary]

class TestSummaryParams(Schema):
  year: Optional[int]
  type: Optional[TestType] = TestType.VESTIBULAR.value

class TestDetailSubject(Schema):
  name: str
  number_of_questions: int
  disciplines: List[Discipline]
  with_essay: Optional[bool] = None

class TestDetailPhases(Schema):
  phase: int
  subjects: List[TestDetailSubject]

class TestDetailOut(Schema):
  id: str
  type: TestType
  year: int
  phases: List[TestDetailPhases]

class QuestionParams(Schema):
  with_answers: bool = False

QuestionOut = List[QuestionDetail]

class QuestionOption(Schema):
  id: str
  statement: str
  letter: str
  is_correct: Optional[bool] = None

class QuestionDetail(Schema):
  id: str
  discipline: List[Discipline]
  type: FormatType
  statement: str
  options: List[QuestionOption]
  images: List[str]
```

**Endpoints:**

| Método e rota | Query params | Payload | Output |
|---------------|--------------|---------|--------|
| GET /api/v1/tests/ | TestSummaryParams | — | ListTestsOut |
| GET /api/v1/tests/{id} | — | — | TestDetailOut |
| GET /api/v1/tests/{id}/questions | QuestionParams | — | QuestionOut |

## Alternatives Considered

| Alternativa | Motivo de não adoção |
|-------------|----------------------|
| Apenas scraping sob demanda (sem ETL e banco) | Não escala; latência e carga no site; difícil padronizar histórico e reutilizar em outras fases (tópicos, simulados). |
| Manter apenas arquivos locais em disco, sem API | Não oferece contrato estável para clientes (apps, estudos); difícil versionar e evoluir (filtros, paginação, gabarito condicional). |
| Incluir escrita (upload de provas) na API nesta fase | Aumenta escopo e segurança; o foco atual é consulta com dados já validados pelo ETL. |
| Reutilizar ou estender a [API do ENEM](https://docs.enem.dev/) | A API do ENEM cobre provas e questões do ENEM (projeto open-source da comunidade); o ITA tem estrutura, formato e público diferentes, justificando uma API dedicada. Serve como referência de desenho (API pública de consulta, documentação, self-hosting). |

## Impact

- **Escopo:** Novo sistema (pipeline ETL + API de leitura); consumo da área de provas do site oficial (leitura/scraping) e possivelmente arquivos locais.
- **Esforço:** Fase 1 depende de implementar os três estágios do ETL e os três endpoints; varia conforme formato das fontes (PDF/HTML) e qualidade dos dados.
- **Riscos:** Mudanças no site do ITA podem quebrar o scraper; mitigação com testes de integração e fallback para arquivos locais quando disponíveis.
- **Rollback:** API e ETL são aditivos; não alteram o sistema legado. Em caso de problema, pode-se desligar a API e manter apenas os JSONs/banco para uso interno.

## Open Questions

- Escolha de stack para o ETL (linguagem, ferramentas de scraping, banco de dados).
- Estratégia de atualização: agendada (cron) vs. sob demanda vs. híbrida.
- Política de versionamento da API (ex.: `/api/v1/` vs. header).
- Como tratar imagens nas questões (URLs, armazenamento, CDN).
- Critérios de qualidade e validação dos dados após a transformação (ex.: checagem de número de questões por disciplina).

## References

- [RFC 0001 - ITA API (este documento)](rfcs/pt/0001-ita-api.md)
- [Programa do Vestibular ITA 2026](https://www.vestibular.ita.br/programa_2026.pdf)
- Área de provas: https://www.vestibular.ita.br/
- [API do ENEM (docs.enem.dev)](https://docs.enem.dev/) – API pública open-source para consulta de provas e questões do ENEM; referência de iniciativa similar em contexto de vestibular brasileiro (licença GNU GPL-2.0, documentação, rate limits, self-hosting).


