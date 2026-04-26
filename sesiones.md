---
type: Note
status: Active
tags: [sesiones, registro]
---

# 📋 Registro de Sesiones

## S-001 — 2026-04-22
- **Tema:** Configuración inicial Fangy + Resend + GitHub
- **Tags:** `wai-88` `resend` `email` `github` `waiprop`
- **Logros:** API key Resend guardada, `fangy@matiasprieto.dev` operativo, crons de reportes waiprop
- **Estado:** ✅ Cerrada

## S-002 — 2026-04-26
- **Tema:** Conexión Mac + Seguridad + Yubrin Deploy + Bóveda + La Juga
- **Tags:** `tailscale` `ssh` `seguridad` `mac` `yubrin` `ansible` `deploy` `traefik` `la-juga` `prefetch` `workers`
- **Logros:**
  - 🔐 La Guanaca blindada (firewall, SSH restringido, sin password auth)
  - 📧 Resend operativo con dominio propio
  - 🔑 Fine-Grained PAT con scope a repos clave
  - 🚀 Yubrin preparado: workers 4→16, timeout 15→10s, concurrent 20→60
  - 📚 Bóveda creada con 6 notas: La Juga, estándar agentes, prefetch, Yubrin scoring, Redis cache, índice
  - 🧉 La Juga documentada: Prefetch + Filter Tree + Pre-contexto
  - 📋 Ansible playbook listo para deploy
- **Pendientes:**
  - ⬜ Deployar Yubrin (`ansible-playbook yubrin-deploy.yml`)
  - ⬜ Implementar La Juga en waiprop_agent_pydantic
  - ⬜ Conectar Yubrin scoring con n8n
  - ⬜ Agregar Redis cache a Yubrin para Hot Sale
- **Estado:** 🔄 En pausa

## S-003 — ■ Pendiente
- **Tema:** [A definir]
- **Tags:**
- **Estado:** ⬜ Pendiente
