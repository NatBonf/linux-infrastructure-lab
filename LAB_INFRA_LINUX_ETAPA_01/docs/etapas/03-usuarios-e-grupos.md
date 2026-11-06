# Etapa 03 — Usuários e grupos no domínio

**Data de conclusão da documentação:** 05/11/2026  
**Ambiente:** laboratório virtualizado em rede interna  
**Domínio:** `lab.internal`  
**Controlador de domínio:** `ubuntu-dc.lab.internal` — `192.168.10.10`  
**Servidor de arquivos:** `debian-fs` — `192.168.10.20`

---

## 1. Objetivo

Criar usuários e grupos de segurança no Active Directory do domínio `LAB.INTERNAL`, organizando as identidades por departamento, e validar se essas contas são reconhecidas pelo servidor Debian `debian-fs`.

O foco desta etapa foi a administração das identidades no controlador de domínio e a verificação de sua resolução no servidor membro. A aplicação de permissões aos diretórios e compartilhamentos não faz parte deste escopo e permanece reservada à Etapa 04.

## 2. Visão geral da infraestrutura

| Componente | Configuração |
| :--- | :--- |
| **Controlador de domínio** | Ubuntu Server — `ubuntu-dc.lab.internal` |
| **Endereço do controlador** | `192.168.10.10` |
| **Servidor de arquivos** | Debian — `debian-fs` |
| **Endereço do servidor de arquivos** | `192.168.10.20` |
| **Domínio DNS** | `lab.internal` |
| **Realm Kerberos** | `LAB.INTERNAL` |
| **Grupo de trabalho NetBIOS** | `LAB` |
| **Rede do laboratório** | `192.168.10.0/24` |

A infraestrutura de domínio e a integração do Debian como servidor membro foram estabelecidas nas etapas anteriores. Nesta fase, o Ubuntu DC foi utilizado para criar e administrar as contas; o Debian FS foi utilizado para consultar e validar as identidades por meio do Winbind.

## 3. Planejamento de usuários e grupos

Foi definida uma estrutura enxuta, com um grupo de segurança e um usuário de teste para cada departamento.

| Departamento | Grupo no domínio | Usuário de teste |
| :--- | :--- | :--- |
| Tecnologia da Informação | `TI` | `usuario.ti` |
| Recursos Humanos | `RH` | `usuario.rh` |
| Financeiro | `FINANCEIRO` | `usuario.fin` |
| Diretoria | `DIRETORIA` | `usuario.dir` |

Os nomes foram definidos para facilitar a identificação das contas durante os testes. Os usuários são fictícios e destinados exclusivamente ao laboratório.

## 4. Criação dos grupos no Ubuntu DC

No controlador de domínio `ubuntu-dc`, foram criados os quatro grupos de segurança:

```bash
sudo samba-tool group add TI
sudo samba-tool group add RH
sudo samba-tool group add FINANCEIRO
sudo samba-tool group add DIRETORIA
```

## 5. Criação dos usuários no Ubuntu DC

Em seguida, foram criadas as quatro contas de teste:

```bash
sudo samba-tool user create usuario.ti '<SENHA_DE_TESTE>' --given-name='Usuario' --surname='TI'
sudo samba-tool user create usuario.rh '<SENHA_DE_TESTE>' --given-name='Usuario' --surname='RH'
sudo samba-tool user create usuario.fin '<SENHA_DE_TESTE>' --given-name='Usuario' --surname='Financeiro'
sudo samba-tool user create usuario.dir '<SENHA_DE_TESTE>' --given-name='Usuario' --surname='Diretoria'
```

O valor `<SENHA_DE_TESTE>` representa a senha definida durante a criação das contas e foi omitido da documentação. Credenciais não devem ser armazenadas no repositório.

## 6. Associação dos usuários aos grupos

Após a criação das contas, cada usuário foi associado ao grupo correspondente:

```bash
sudo samba-tool group addmembers TI usuario.ti
sudo samba-tool group addmembers RH usuario.rh
sudo samba-tool group addmembers FINANCEIRO usuario.fin
sudo samba-tool group addmembers DIRETORIA usuario.dir
```

Essa associação estabelece a organização departamental das identidades no Active Directory. A aplicação de permissões com base nesses grupos será realizada na etapa seguinte.

## 7. Validação das identidades no Debian FS

Com os usuários e grupos criados no controlador, a validação foi realizada no servidor `debian-fs`, por meio das ferramentas Winbind e NSS.

### 7.1 Enumeração de usuários

```bash
wbinfo -u
```

As quatro contas criadas foram retornadas pelo Winbind:

```text
LAB\usuario.ti
LAB\usuario.rh
LAB\usuario.fin
LAB\usuario.dir
```

O resultado confirmou que o Debian conseguiu enumerar os usuários do domínio.

### 7.2 Enumeração de grupos

```bash
wbinfo -g
```

Os grupos departamentais foram identificados:

```text
LAB\ti
LAB\rh
LAB\financeiro
LAB\diretoria
```

A consulta confirmou que os grupos criados no Active Directory também estavam disponíveis para o servidor membro.

### 7.3 Resolução dos usuários e mapeamento de IDs

Para verificar a resolução das identidades pelo sistema, foi utilizado o comando `id` para cada usuário:

```bash
id "LAB\usuario.ti"
id "LAB\usuario.rh"
id "LAB\usuario.fin"
id "LAB\usuario.dir"
```

Os resultados observados foram:

| Usuário | UID | GID primário | Grupo departamental (GID suplementar) |
| :--- | :--- | :--- | :--- |
| `LAB\usuario.ti` | 11108 | 10513 (`domain users`) | `LAB\ti` — 11104 |
| `LAB\usuario.rh` | 11109 | 10513 (`domain users`) | `LAB\rh` — 11105 |
| `LAB\usuario.fin` | 11110 | 10513 (`domain users`) | `LAB\financeiro` — 11106 |
| `LAB\usuario.dir` | 11111 | 10513 (`domain users`) | `LAB\diretoria` — 11107 |

Os quatro usuários foram resolvidos pelo Debian. Em todos os resultados, o grupo `LAB\domain users` aparece como grupo primário, enquanto o grupo departamental correspondente aparece entre os grupos suplementares.

O comando id confirmou a associação dos usuários aos grupos departamentais, além dos grupos LAB\domain users e BUILTIN\users. Também foram identificados grupos individuais associados às contas.

### 7.4 Consulta dos grupos pelo NSS

A resolução dos grupos foi verificada com:

```bash
getent group "LAB\ti"
getent group "LAB\rh"
getent group "LAB\financeiro"
getent group "LAB\diretoria"
```

Resultados observados:

```text
LAB\ti:x:11104:
LAB\rh:x:11105:
LAB\financeiro:x:11106:
LAB\diretoria:x:11107:
```

As consultas retornaram os quatro grupos e seus respectivos GIDs. A associação individual dos usuários aos grupos departamentais foi confirmada pelos resultados do comando `id`.

### 7.5 Verificação do intervalo de mapeamento

Os UIDs e GIDs apresentados ficaram dentro do intervalo configurado para o domínio `LAB` no arquivo `smb.conf` do Debian FS, definido na Etapa 02:

```ini
idmap config LAB : backend = rid
idmap config LAB : range = 10000-99999
```

Os identificadores retornados são compatíveis com o mapeamento realizado pelo backend `rid` e com o intervalo configurado.

## 8. Resultado da etapa

Ao final desta etapa, foram realizados e verificados os seguintes itens:

* Quatro grupos de segurança criados no Active Directory.
* Quatro usuários de teste criados no domínio.
* Associação de cada usuário ao respectivo grupo departamental.
* Emuneração dos usuários pelo Winbind no Debian FS.
* Enumeração dos grupos pelo Winbind no Debian FS.
* Resolução dos quatro usuários pelo comando `id`.
* Consulta dos grupos pelo NSS com `getent`.
* UIDs e GIDs identificados dentro do intervalo de mapeamento configurado.

**Situação:** a criação das identidades no domínio e sua resolução no Debian FS foram validadas. Os grupos departamentais estão disponíveis para serem utilizados na próxima etapa, que tratará da definição de permissões e do controle de acesso aos diretórios e compartilhamentos.

## 9. Próximas etapas e planejamento

A sequência planejada para continuidade do laboratório é:

1. **Etapa 04 — Permissões:** definição de permissões Linux, ACLs e permissões SMB, seguida de testes com diferentes identidades.
2. **Etapa 05 — Segurança:** hardening, auditoria e testes controlados.
3. **Etapa 06 — Consolidação:** testes integrados, diagrama e documentação geral do laboratório.

---

## Observações de documentação

Esta etapa não exigiu alterações na configuração de rede, no ingresso do Debian ao domínio ou nos parâmetros de integração Samba/Winbind estabelecidos anteriormente. O trabalho concentrou-se na criação das identidades no Ubuntu DC e na validação de sua resolução no Debian FS.

Os resultados documentam a enumeração e o mapeamento de usuários e grupos. Eles não representam, por si só, a configuração ou validação de permissões sobre os diretórios e compartilhamentos. Esse controle permanece pendente para a Etapa 04.