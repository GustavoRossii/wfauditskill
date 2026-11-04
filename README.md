# wfaudit

Skill do Claude Code que roda um workflow completo de auditoria de projeto: superfície de ataque, segurança, bugs de lógica, verificação de falsos positivos, testes, arquitetura e organização. Gera um `AUDITORIA.md` e só aplica correções com aprovação.

## Instalar

Copie a pasta `wfaudit` para a pasta de skills do Claude Code.

Windows (PowerShell):

```powershell
git clone https://github.com/GustavoRossii/wfauditskill.git
Copy-Item -Recurse wfauditskill\wfaudit $env:USERPROFILE\.claude\skills\wfaudit
```

Linux/macOS:

```bash
git clone https://github.com/GustavoRossii/wfauditskill.git
cp -r wfauditskill/wfaudit ~/.claude/skills/wfaudit
```

## Usar

No Claude Code, dentro do projeto:

```
/wfaudit
/wfaudit foco opcional, por exemplo: módulo de pagamentos
```

## Dependências

A skill chama outras skills (`cso`, `security-reviewer`, `insecure-defaults`, `fp-check`, `entry-point-analyzer`, `trailmark`, `static-analysis` etc.). As que não estiverem instaladas são puladas, com um aviso. Quanto mais skills instaladas, mais completa a auditoria.
