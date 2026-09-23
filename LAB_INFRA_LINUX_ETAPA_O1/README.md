# Laboratório de Infraestrutura Linux

Laboratório prático de infraestrutura de redes e serviços Linux, desenvolvido em máquinas virtuais e documentado por etapas.

O objetivo deste repositório é registrar a evolução do ambiente desde a implantação do primeiro serviço de infraestrutura até uma estrutura mais completa, envolvendo domínio, usuários, permissões, compartilhamento de arquivos, administração e segurança.

> **Status:** Etapa 01 concluída — Controlador de Domínio Samba AD/DC no Ubuntu Server.

## Arquitetura inicial

| Componente | Função | Estado |
|---|---|---|
| Ubuntu Server | Controlador de Domínio / DNS / Kerberos | ✅ Etapa 01 |
| Debian | File Server | 🔜 Próxima etapa |
| Kali Linux | Laboratório de segurança | 🔜 Futuro |
| Rede interna | Ambiente isolado de laboratório | ✅ |

### Domínio

- Hostname: `ubuntu-dc`
- FQDN: `ubuntu-dc.lab.internal`
- Domínio: `lab.internal`
- Realm Kerberos: `LAB.INTERNAL`
- NetBIOS: `LAB`
- IP do DC: `192.168.10.10`

## Etapas planejadas

1. **Samba AD/DC no Ubuntu** — domínio, DNS e Kerberos.
2. **File Server Debian** — ingresso no domínio e estrutura de compartilhamentos.
3. **Usuários e grupos** — organização de identidades.
4. **Permissões** — ACLs, compartilhamentos e acesso por grupos.
5. **Administração** — rotinas de gerenciamento e validação.
6. **Segurança** — hardening, testes e observações de segurança.
7. **Integração e testes** — validação do ambiente como um todo.
8. **Documentação final** — consolidação das etapas em um único documento técnico.

## Estrutura do repositório

```text
lab-infra-linux/
├── README.md
├── docs/
│   └── etapas/
│       └── 01-samba-ad-dc.md
