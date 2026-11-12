# Prompt para o Claude Code — Auditoria da checklist AT (só leitura)

**Objetivo:** produzir um relatório de lacunas entre o estado atual do WashInvoice POS e os requisitos de certificação AT. **Esta tarefa NÃO altera código.** O resultado é um ficheiro `AUDITORIA_AT.md` na raiz do projeto.

## Contexto
POS + faturação para lavandarias, Windows, Flutter + Drift/SQLite, a aproximar-se da v15. Tabelas confirmadas na BD: `empresa` (inclui `logotipo_path`), `utilizadores` (PIN com hash), `series`, `documentos_fiscais`, campos ATCUD, audit logs. Não assumas a existência de mais nenhuma tabela, coluna ou ficheiro: **se não o encontrares no código, regista como "não encontrado"**, nunca o inventes.

## Passo 0 — Mapear
Lista as tabelas Drift reais (ficheiro de schema/`.drift`/tabelas Dart), os DAOs e os serviços que tocam em documentos fiscais, séries, hash, ATCUD, QR e SAF-T. Indica os caminhos reais dos ficheiros antes de avaliar.

## Passo 1 — Avaliar cada requisito (✅ cumpre / ⚠️ parcial / ❌ falta / ❓ não encontrado), com ficheiro:linha como prova

1. **ATCUD**
   - Formato `CódigoValidaçãoSérie-NºSequencial`, presente em todos os documentos fiscais, impresso e no QR.
   - Origem do código de validação da série (campo guardado em `series`?) e o que acontece se estiver vazio.
2. **Hash encadeado (RSA)**
   - Algoritmo (RSA-SHA1 sobre: data documento; data/hora sistema; nº documento; total bruto; hash anterior), hash do 1.º documento da série, impressão dos 4 carateres (posições 1, 11, 21, 31 do hash).
   - Gestão de chaves: onde vive a chave privada, como é protegida, versão da chave.
   - Impossibilidade de alterar/apagar documentos emitidos (sem UPDATE/DELETE em `documentos_fiscais`; anulação só por documento retificativo).
3. **Numeração e séries**
   - Sequência sem falhas por série, atómica (transação), sem reutilização. Teste de concorrência/crash a meio da emissão.
   - Séries por tipo de documento; comunicação à AT (webservice/Portal) — está implementada, manual ou em falta?
4. **QR Code** (especificação AT em vigor: campos A–S, NIF emitente/adquirente, país, tipo, estado, data, ATCUD, bases/IVA por taxa, total, 4 carateres do hash, nº certificado). Verifica campos obrigatórios e o tamanho/qualidade de impressão no talão do SAM4S.
5. **SAF-T (PT)**
   - Valida o XML gerado contra o XSD oficial em vigor (não assumas a versão: indica a que o código usa e a vigente). Cabeçalho, `MasterFiles`, `SourceDocuments`, totais, `HashControl`, `SystemEntryDate`.
   - Executa uma exportação com dados de teste e valida com o XSD.
6. **IVA e totais**: arredondamentos, taxas, motivos de isenção (M-codes), consistência SAF-T ↔ documento impresso ↔ apuramento de IVA.
7. **Documentos impressos**: menção obrigatória (nome do programa + nº certificado "Processado por programa certificado nº X/AT" — hoje será placeholder), cópias/segundas vias marcadas, NIF, dados de `empresa`.
8. **Fronteira fiscal vs. interno**: confirmar que movimentos de caixa (fundo, reforço, sangria, despesa, diferença, fecho) **não** geram ATCUD/hash/série, mas ficam com timestamp + utilizador + motivo. Testar a regra: *total faturado em dinheiro = sangrias + fecho − fundo inicial*. Confirmar que o consumo de créditos/cabides e ofertas de cortesia são internos e que a **venda** do crédito é o evento fiscal.
9. **Auditoria e segurança**: audit logs cobrem emissão, anulação, alteração de séries, exportações, mudanças de utilizador/PIN; PIN em hash; relógio do sistema (deteção de recuo de data); acesso a ecrãs `saft` só com perfil adequado.
10. **Cópias de segurança**: o backup preserva a cadeia de hash e permite restauro sem quebrar a numeração.

## Passo 2 — Testes reais
Onde possível, corre a app/testes e emite pelo caminho real da UI (não apenas teste unitário): fatura, fatura-recibo, nota de crédito, anulação, exportação SAF-T. Regista o resultado; se não conseguires correr a UI, di-lo explicitamente.

## Passo 3 — Entregável `AUDITORIA_AT.md`
- Tabela: requisito | estado | prova (ficheiro:linha) | risco | esforço (S/M/L).
- Lista priorizada de lacunas **bloqueantes** da submissão, depois as não bloqueantes.
- Perguntas em aberto para o Cesar (coisas que dependem de decisão ou do Portal das Finanças).
- Nota final: confirmar no Portal das Finanças os requisitos vigentes antes de dar algo por fechado.

Textos em português europeu.

## NÃO TOCAR EM:
- Qualquer código-fonte, schema, migrações ou dados (tarefa estritamente de leitura; só cria `AUDITORIA_AT.md`).
- Lógica de negócio, RouterGate, licenciamento (`licenca.json`, `_hmacKey`, machine_id), ecrãs de UI.
- Ficheiros não mencionados acima.
- Não propor encriptação da BD (SQLCipher rejeitado deliberadamente).
