# Dashboard de Cobertura — Oficinas & Benefício da Apólice

Dashboard interativo que substitui a consulta manual à planilha. Permite **adicionar e
remover oficinas** e recalcula **automaticamente** quais cidades ficam elegíveis ao
benefício, segundo a regra de **raio de 50 km** da oficina.

## Como usar

Abra o arquivo **`index.html`** no navegador (duplo clique). Não precisa de servidor nem
instalação — funciona totalmente offline.

### Fluxo típico (cliente reclamou)

1. Vá ao painel **🔎 Verificador de cobertura do cliente** (rodapé).
2. Digite a cidade do cliente e clique em **Verificar**.
3. O sistema informa se a cidade está no raio de 50 km de alguma oficina ativa, quais
   oficinas a cobrem e se ela já consta no sistema.

## Funcionalidades

| Recurso | O que faz |
|---|---|
| **KPIs** | Oficinas ativas, cidades elegíveis, cidades a incluir/retirar e estados atendidos. |
| **Oficinas cadastradas** | Lista com busca. Ative/desative pelo botão e remova pela lixeira. |
| **Adicionar oficina** | Informe nome, CEP, cidade/UF e as cidades atendidas (raio de 50 km). |
| **Atualizações no sistema** | Compara as cidades elegíveis com as já cadastradas e mostra, por estado, o que **incluir** (passou a ter cobertura) e o que **retirar** (deixou de ter oficina no raio). |
| **Cidades elegíveis** | Lista, por estado, todas as cidades cobertas por alguma oficina ativa. |
| **Verificador de cliente** | Consulta rápida por cidade. |
| **Carregar planilha** | Importa a planilha base (`.xlsx`) com as abas **Regiões** e **Sistemas**. |
| **Exportar** | Baixa um CSV com as ações (incluir/retirar) e a lista de cidades elegíveis. |

Toda alteração é recalculada em tempo real e salva **no próprio navegador**
(`localStorage`). Use **↺ Restaurar** para descartar as alterações e voltar à base.

## Regra de cobertura (raio de 50 km)

Cada oficina carrega a sua **área de abrangência** — a lista de municípios dentro do raio
de ~50 km, conforme a planilha da Porto (coluna *Regiões de Abrangência*). Uma cidade é
**elegível** quando pelo menos **uma oficina ativa** a cobre. Ao ativar, desativar,
adicionar ou remover uma oficina, a união das áreas é recalculada automaticamente e as
listas de inclusão/remoção são atualizadas.

Ao adicionar uma nova oficina, informe as cidades dentro do raio de 50 km no campo
*Cidades atendidas*.

## Estrutura da planilha base

- **Aba `Regiões`**: oficinas da Porto. Colunas usadas: `CAPS` (nome),
  `REGIÕES DE ABRANGÊNCIA` (cidades no raio), `CIDADE`, `ESTADO`, `CEP`.
- **Aba `Sistemas`**: cidades já cadastradas no sistema, por `Estado` e `Cidades`.

O leitor de planilha reconhece as colunas pelos títulos, então pequenas variações de
posição são toleradas.

## Arquivos

```
index.html               # o dashboard (dados base embutidos)
vendor/xlsx.full.min.js  # leitor de .xlsx (SheetJS) — para funcionar offline
```
