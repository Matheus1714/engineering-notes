# Title

> **Dica:** Um título claro e específico. Ex.: "API interna (ITA) para comunicação entre serviços".

## Summary

> **O que é:** 2–3 frases que qualquer pessoa entenda. O que estamos propondo e qual o resultado esperado.
>
> **Boa questão:** *What real problem does this solve?*

Resumo em uma frase: estamos criando uma API interna (ITA) para que os microserviços se comuniquem de forma padronizada e auditável, substituindo chamadas HTTP ad hoc e reduzindo acoplamento.

## Motivation / Problem Statement

> **O que é:** Por que isso importa agora? Qual dor/oportunidade estamos endereçando?
>
> **Boas questões:** *What real problem does this solve?* | *Why now?*

Hoje cada time expõe endpoints HTTP com contratos diferentes, sem documentação centralizada e com autenticação inconsistente. Isso gera retrabalho, bugs de integração e dificulta onboarding. A decisão de padronizar agora vem do crescimento do número de serviços (de 5 para 15 no último ano) e de incidentes recorrentes de integração.

## Goals / Non-Goals

> **O que é:** O que entra no escopo (goals) e o que explicitamente fica de fora (non-goals). Evita escopo creep.
>
> **Boas questões:** *What changes?* | *What doesn't change?*

**Goals:**

- Definir contrato único (OpenAPI) e gateway para todas as chamadas serviço-a-serviço.
- Autenticação mútua (mTLS) e logging de todas as chamadas para auditoria.
- Documentação e SDK por linguagem (inicialmente Go e Python).

**Non-Goals:**

- Não substituir APIs públicas ou de parceiros.
- Não incluir filas/eventos nesta fase; apenas request/response síncrono.

## Proposed Solution

> **O que é:** A solução concreta: arquitetura, fluxos, decisões técnicas principais. Suficiente para implementar sem ambiguidade.
>
> **Boas questões:** *What changes?* | *What are the risks?*

- **Gateway:** Um único ingress (ex.: Kong ou similar) na rede interna; todos os serviços registram rotas e consomem via esse gateway.
- **Contrato:** OpenAPI 3.0 obrigatório; geração de clientes a partir do spec.
- **Auth:** mTLS com certificados emitidos por PKI interna; rotação automática.
- **Observabilidade:** Todas as requisições logadas (service, path, status, latency); métricas no mesmo sistema já usado hoje.

Fases: (1) Piloto com 2 serviços, (2) Migração gradual dos demais, (3) Deprecação de chamadas diretas.

## Alternatives Considered

> **O que é:** Outras opções que foram avaliadas e por que foram descartadas. Mostra que a decisão foi pensada.
>
> **Boa questão:** *How do we undo this if things go wrong?* (alternativas são “planos B” implícitos)

| Alternativa | Por que não |
|-------------|-------------|
| Cada serviço continuar expondo HTTP direto | Não resolve inconsistência de contrato, auth e observabilidade. |
| Usar apenas service mesh (ex.: Istio) | Custo e complexidade altos para o estágio atual; mesh pode vir depois em cima do gateway. |
| API REST única monolítica | Rejeitado; queremos serviços independentes, apenas com contrato e gateway comuns. |

## Impact

> **O que é:** Quem é afetado (times, sistemas, usuários), esforço estimado e dependências.
>
> **Boas questões:** *What changes?* | *What doesn't change?* | *What are the risks?*

- **Times:** Todos os times com serviços que chamam outros internamente; estimativa de 2–3 sprints para migração total.
- **Sistemas:** Novo componente (gateway + PKI); mudança em como os serviços se registram e fazem chamadas.
- **Riscos:** Curva de aprendizado do mTLS; mitigação com documentação e piloto pequeno.
- **Rollback:** Gateway pode ser configurado para encaminhar ao antigo endpoint por serviço; migração é reversível por serviço.

## Open Questions

> **O que é:** Pontos ainda não decididos que precisam de resposta antes ou durante a implementação.

- Escolha final do gateway (Kong vs outro) após POC de desempenho.
- Política de versionamento da API (path vs header vs novo host).
- Prazo para deprecação de chamadas diretas após migração (3 ou 6 meses).

## References

> **O que é:** Links para docs, RFCs, ADRs ou discussões que fundamentam a proposta.

- [RFC 0001 - ITA API](rfcs/0001-ita-api.md) (este documento)
- Especificação OpenAPI 3.0
- Documentação interna de PKI e mTLS

---

## Good Questions (checklist para o RFC)

Antes de fechar o RFC, verifique se estas perguntas estão respondidas em alguma seção:

- **What real problem does this solve?** → Summary, Motivation
- **Why now?** → Motivation
- **What changes?** → Goals, Proposed Solution, Impact
- **What doesn't change?** → Goals (Non-Goals), Impact
- **What are the risks?** → Proposed Solution, Impact, Alternatives
- **How do we undo this if things go wrong?** → Impact (rollback), Alternatives
