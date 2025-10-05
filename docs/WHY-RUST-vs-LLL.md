Aqui vai um texto direto, para virar docs/WHY-RUST-vs-LLL.md (ou colar no README). Ele fixa a filosofia, as fronteiras e o que é aceitável em cada lado — sem meia-palavra.

⸻

Por que Rust e por que .lll (e quando usar cada um)

0) Verdade fundadora

LogLineOS é um experimento deliberado para provar o poder do .lll.
Se a lógica de negócio/fluxo/regra não estiver em .lll, estamos falhando o experimento. A escolha por Rust é apenas para o que precisa ser Trusted Computing Base (TCB): isolamento, verificação criptográfica, limites de recurso, e a ponte segura para o mundo externo.

Resumo em 1 linha: .LLL é protagonista. Rust é guarda-costas.

⸻

1) Princípios
1. .LLL-first: tudo que expressa intenção, regra, política, derivação, roteamento, qualidade deve ser .lll.
2. TCB mínimo: Rust só entra onde segurança, isolamento ou desempenho exigem — e nunca codifica regra de domínio.
3. Determinismo e auditabilidade: regras .lll são executáveis, versionáveis, auditáveis, e reproduzíveis com receipts.
4. Multi-tenant por design: cada decisão carrega tenant, roles e capabilities, definidos em .lll.
5. Sem atalhos: qualquer “atalho” de regra no Rust contamina o core e é tratado como bug.

⸻

2) O que DEVE ser Rust (TCB)
• Loader & Sandbox: validação de .lllb assinado, isolamento de execução, deny-by-default.
• Hostcalls ABI: fronteiras para ledger.append, kv, http, ws, crypto.verify, com capability gating.
• Kernels quentes: primitivas críticas (p.ex. ledger.append de alta taxa, verificação ed25519).
• Identidade & Segurança: OIDC/JWT (LLST), JWKS, rotação de chaves, rate-limit/quotas no fio.
• Métricas de base e limites de recurso: cgroups/rlimits, medição de latência/erros, backpressure do gateway.

Teste de bolso: se tirar a internet e rodar offline, o Rust do TCB ainda protege a execução? Se “não”, isso era .lll.

⸻

3) O que DEVE ser .lll (protagonista)
• Roteamento & validação: canonical_contract_v1, PII redaction, regras de ingest.
• Derivação neutra: trajectory_linker, trajectory_close, geração de trajectory_edge/closed.
• Qualidade: trajectory_quality_score + quality_meter_v1_1 (emite diamond_candidate neutro).
• Policies & quotas: pesos, percentis, janelas, limites por tenant/role.
• Observabilidade lógica: nomes de métricas, contadores públicos, envelopes de qualidade.
• Extensões de app: qualquer lógica de domínio (Lab/Powerfarm/Minicontratos, Padaria/Produtora) vive do lado do app, em .lll, nunca no TCB.

Teste de bolso: esta decisão muda “o que” acontece ou “quando/como” acontece no negócio? Então é .lll.

⸻

4) Matriz de decisão (copiável)

PerguntaVai para…
Precisa de isolamento, assinatura, limite de CPU/mem?Rust (TCB)
É regra de pipeline (ligar, derivar, filtrar, pontuar)?.LLL
É acesso a recurso externo (rede/FS) com política de permissão?Rust hostcall + policy .lll
É mapeamento de tokens (OIDC/JWT) e verificação de chave?Rust (TCB)
É “o que é um candidato”, “como linka trajetória”? .LLL
É contador/alarme “quando passar de X”? .LLL (regra) + Rust (exposição métrica)

⸻

5) Exemplos mínimos

5.1 Regra em .lll (certo)

validator trajectory_quality_score {
  input: "trajectory_edge|trajectory_closed"
  emit: "trajectory_quality" with components(mass,persistence,verification)
}

validator quality_meter_v1_1 {
  input: "trajectory_quality"
  when percentile(score) >= policy.quality_meter.percentile_target
   and score >= policy.quality_meter.min_score
  emit: "diamond_candidate"
}

5.2 Ponte em Rust (certo)

// Hostcall: append-only, sem semântica de domínio
pub fn ledger_append(bytes: &[u8], tenant: Tenant, caps: Caps) -> Result<ReceiptId> {
    enforce_caps(caps, "ledger.append")?;
    // ... validação de tamanho, fsync por lote, receipt ...
    Ok(receipt_id)
}

5.3 “Regra” no Rust (errado)

// ❌ NÃO PODE: lógica de score/derivação/diamante no TCB
fn compute_diamond_score(traj: &Trajectory) -> u32 { /* ... */ }

⸻

6) Anti-contaminação (mecanismos práticos)
• Guard CI: reprovar PR se aparecer trajectory|quality|diamond|fold|fto em arquivos .rs fora da lista branca do TCB.
• CODEOWNERS: core/logline-core/src/** = time runtime; /pipelines/** e /validators/** = time de regras.
• Pre-commit: grep nos diffs procurando padrões proibidos em Rust.
• PR template: checkboxes “Regra em .lll?”, “Semântica fora do TCB?”, “Receipts/metrics ok?”.

⸻

7) Por que isso é importante (técnico e filosófico)
• Composição & replay: .lll gera pipelines reexecutáveis com receipts; regra em Rust vira caixa preta.
• Governança multi-tenant: políticas mutáveis por tenant sem rebuild de binário.
• Seguro de projeto: trocar peso de score, janela de trajetória, percentil — tudo sem redeploy.
• Velocidade com segurança: Rust protege e acelera o que não pode falhar; .lll acelera o que precisa evoluir.

Conclusão dura: esconder regra de negócio no Rust é desserviço ao LogLineOS. O objetivo deste sistema é provar que .lll carrega a lógica com auditabilidade, reprodutibilidade e poder de composição — e que o Rust fica onde deve: TCB mínimo, rápido e seguro.

⸻

8) Quando abrir exceção?

Quase nunca. Se um trecho de .lll for provavelmente inviável (p.ex., um kernel criptográfico específico), escreva em Rust como hostcall puro, documente a razão, não coloque semântica, e exponha capability + policy para controlar uso. Toda exceção deve vir com:
• justificativa técnica (bench/segurança),
• alternativa em .lll (mesmo que lenta) para referência,
• plano de revisão futura.

⸻

9) Frase de contrato (para afixar no repo)

Contrato do Core: “Se é regra, é .lll. Se é kernel, é Rust. Se é dúvida, vira .lll até prova em contrário.”

⸻
