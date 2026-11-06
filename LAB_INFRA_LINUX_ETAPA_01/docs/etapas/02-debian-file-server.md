# Etapa 02 — Servidor de Arquivos Debian (Debian-FS)

**Data de conclusão da documentação:** 03/11/2026  
**Ambiente:** laboratório virtualizado em rede interna  
**Domínio:** `lab.internal`  
**Controlador de domínio:** `ubuntu-dc.lab.internal` — `192.168.10.10`  
**Servidor de arquivos:** `debian-fs` — `192.168.10.20`

---

## 1. Objetivo

Integrar um servidor Debian ao domínio `LAB.INTERNAL`, preparar uma estrutura inicial de diretórios e publicar compartilhamentos SMB por meio do Samba.

Nesta etapa, o foco foi estabelecer a comunicação entre o servidor de arquivos e o controlador de domínio, disponibilizar os compartilhamentos e verificar seu acesso. A definição de usuários, grupos e permissões específicas fica para as etapas seguintes.

## 2. Visão geral da infraestrutura

| Componente | Configuração |
|---|---|
| Controlador de domínio | Ubuntu Server — `ubuntu-dc.lab.internal` |
| Endereço do controlador | `192.168.10.10` |
| Servidor de arquivos | Debian — `debian-fs` |
| Endereço do servidor de arquivos | `192.168.10.20` |
| Domínio DNS | `lab.internal` |
| Realm Kerberos | `LAB.INTERNAL` |
| Grupo de trabalho NetBIOS | `LAB` |
| Rede do laboratório | `192.168.10.0/24` |

As máquinas virtuais foram conectadas à mesma rede interna do VirtualBox (`intnet`). O Debian utiliza o controlador de domínio como servidor DNS.

## 3. Recuperação da conectividade e preparação do Debian

Após um travamento forçado do hipervisor, o Debian iniciou com uma configuração de rede incompatível com a sub-rede planejada. A interface `eth0` estava utilizando um endereço da faixa `192.168.1.0/24`, enquanto o controlador de domínio permanecia na rede `192.168.10.0/24`.

A configuração foi ajustada em `/etc/network/interfaces`, atribuindo ao Debian o endereço estático `192.168.10.20/24`. Também foi necessário alinhar os adaptadores das duas máquinas virtuais à mesma rede interna do VirtualBox.

Durante a preparação, foi aberta temporariamente uma janela de manutenção em modo NAT para instalar os utilitários necessários. Após a instalação, o adaptador foi devolvido à rede interna.

Pacotes utilizados nesta etapa:

```bash
sudo apt update
sudo apt install smbclient cifs-utils chrony -y
```

O `smbclient` foi utilizado para consultar e testar os compartilhamentos. O `cifs-utils` foi instalado como suporte a operações com sistemas de arquivos CIFS/SMB.

## 4. Sincronização de horário

A diferença entre os relógios das máquinas virtuais provocou falhas intermitentes durante as tentativas de autenticação Kerberos. O horário foi alinhado para que o Debian utilizasse o Ubuntu DC como referência NTP por meio do Chrony.

No Debian, a fonte configurada foi:

```text
server 192.168.10.10 iburst
```

Durante o diagnóstico, a sincronização inicialmente não recebia respostas válidas. Após ajustes no controlador de domínio e na comunicação do Chrony com o mecanismo de assinatura temporal do Samba, a fonte passou a ser selecionada (`*`) e o comando `chronyc tracking` apresentou `Leap status: Normal`.

O registro detalhado das tentativas e dos ajustes de AppArmor foi mantido no diário de troubleshooting. Para a documentação principal desta etapa, o ponto relevante é que o alinhamento temporal foi restabelecido e a sincronização do Debian com o Ubuntu DC foi validada.

## 5. Ingresso do Debian no domínio

Com conectividade, resolução DNS e horário ajustados, foi realizado o ingresso do servidor no domínio:

```bash
sudo net ads join -U Administrator
```

O comando retornou:

```text
Using short domain name -- LAB
Joined 'DEBIAN-FS' to dns domain 'lab.internal'
```

A associação foi verificada com:

```bash
net ads testjoin
```

Resultado:

```text
Join is OK
```

O serviço Winbind foi habilitado e iniciado:

```bash
sudo systemctl enable --now winbind
```

Em seguida, a consulta às contas do domínio foi realizada com:

```bash
wbinfo -u
```

Resultado observado:

```text
LAB\guest
LAB\krbtgt
LAB\administrator
```

Essa consulta confirmou que o Winbind conseguiu enumerar contas do domínio. Ela não representa, isoladamente, uma validação de permissões de acesso aos diretórios.

## 6. Configuração do Samba no Debian-FS

O Debian foi configurado como membro do domínio (`security = ADS`). A configuração global validada pelo `testparm -s` inclui:

```ini
[global]
    client min protocol = SMB3
    client signing = required
    realm = LAB.INTERNAL
    security = ADS
    workgroup = LAB
    idmap config LAB : backend = rid
    idmap config LAB : range = 10000-99999
    idmap config * : backend = tdb
    idmap config * : range = 3000-7999
```

Durante a revisão da configuração, foram corrigidos dois pontos:

- Foi definido um intervalo de mapeamento de identificadores para o domínio `LAB`.
- Foi removida a diretiva `password server`, pois o Samba pode localizar automaticamente o controlador de domínio quando utiliza `security = ADS`.

A validação da configuração foi feita com:

```bash
testparm -s
```

O resultado confirmou que o arquivo de configuração foi carregado. O aviso sobre suporte a criptografia fraca do GnuTLS permaneceu visível e não foi tratado nesta etapa; poderá ser reavaliado durante a etapa de segurança.

## 7. Estrutura de diretórios

Foi criada a estrutura inicial de armazenamento em `/srv/dados`:

```text
/srv/dados/
├── publico/
├── departamentos/
│   ├── diretoria/
│   ├── financeiro/
│   ├── rh/
│   └── ti/
└── comum/
```

A estrutura foi criada com:

```bash
sudo mkdir -p /srv/dados/publico \
    /srv/dados/comum \
    /srv/dados/departamentos/{ti,rh,financeiro,diretoria}
```

Nesta fase, os diretórios permaneceram sob propriedade `root:root`, com permissões `755`.

```bash
sudo chown -R root:root /srv/dados
sudo chmod 755 /srv/dados
sudo chmod 755 /srv/dados/comum
sudo chmod 755 /srv/dados/departamentos
sudo chmod 755 /srv/dados/publico
```

As permissões iniciais permitem leitura e travessia aos demais usuários, mas somente o proprietário `root` possui permissão de escrita. Essa configuração foi mantida intencionalmente para não antecipar a política de acesso que será definida nas próximas etapas.

## 8. Publicação dos compartilhamentos

Foram adicionadas três definições ao arquivo `/etc/samba/smb.conf`:

```ini
[publico]
    path = /srv/dados/publico
    read only = no
    browsable = yes

[departamentos]
    path = /srv/dados/departamentos
    read only = no
    browsable = yes

[comum]
    path = /srv/dados/comum
    read only = no
    browsable = yes
```

Antes de aplicar a configuração, foi realizada a validação com `testparm -s`. O serviço `smbd` foi recarregado no próprio Debian-FS:

```bash
sudo systemctl reload smbd
```

Os compartilhamentos publicados são:

| Compartilhamento | Caminho no servidor | Caminho SMB |
|---|---|---|
| `publico` | `/srv/dados/publico` | `\\DEBIAN-FS\publico` |
| `departamentos` | `/srv/dados/departamentos` | `\\DEBIAN-FS\departamentos` |
| `comum` | `/srv/dados/comum` | `\\DEBIAN-FS\comum` |

## 9. Testes e resultados

### 9.1 Listagem dos compartilhamentos

A consulta local foi executada com:

```bash
smbclient -L localhost -U 'LAB\administrator'
```

O resultado apresentou os compartilhamentos `publico`, `departamentos` e `comum`, além do serviço `IPC$`.

### 9.2 Listagem do conteúdo

Os três compartilhamentos foram consultados individualmente:

```bash
smbclient //localhost/publico -U 'LAB\administrator' -c 'ls'
smbclient //localhost/departamentos -U 'LAB\administrator' -c 'ls'
smbclient //localhost/comum -U 'LAB\administrator' -c 'ls'
```

As consultas foram concluídas. O compartilhamento `departamentos` apresentou as pastas `rh`, `financeiro`, `diretoria` e `ti`; os compartilhamentos `publico` e `comum` estavam vazios.

### 9.3 Teste de escrita

Foi criado um arquivo local para testar o envio ao compartilhamento `publico`:

```bash
echo "Teste de compartilhamento Samba - Etapa 02" > /tmp/teste-samba.txt

smbclient //localhost/publico -U 'LAB\administrator' -c 'put /tmp/teste-samba.txt teste-samba.txt'
```

O envio retornou:

```text
NT_STATUS_ACCESS_DENIED opening remote file \teste-samba.txt
```

A investigação das permissões mostrou que `/srv/dados/publico` permanecia como `root:root`, com modo `755`. Portanto, embora a definição do compartilhamento permita escrita na camada do Samba (`read only = no`), as permissões do sistema de arquivos não concedem escrita ao usuário mapeado utilizado no teste.

O resultado foi mantido como evidência do comportamento atual, sem ampliar as permissões de forma indiscriminada. Não foi utilizado `chmod 777` nem alterada a propriedade dos diretórios para contornar o bloqueio.

## 10. Resultado da etapa

Ao final desta etapa, foram realizados e verificados os seguintes itens:

- Debian configurado na rede interna do laboratório com IP estático.
- Comunicação e resolução de nomes entre Debian-FS e Ubuntu-DC.
- Sincronização de horário do Debian com o controlador de domínio.
- Ingresso do `DEBIAN-FS` no domínio `LAB.INTERNAL`.
- Winbind ativo e capaz de enumerar contas do domínio.
- Estrutura inicial de diretórios criada em `/srv/dados`.
- Três compartilhamentos Samba publicados e listáveis.
- Teste de escrita executado, com bloqueio coerente com as permissões atuais do filesystem.

**Situação:** a publicação e a listagem dos compartilhamentos foram validadas. A política de escrita e o controle de acesso por usuário e grupo permanecem pendentes, conforme o escopo das próximas etapas.

## 11. Próximas etapas e planejamento

A sequência planejada para continuidade do laboratório é:

1. **Etapa 03 — Identidades:** criação e organização de usuários e grupos no domínio.
2. **Etapa 04 — Permissões:** definição de permissões Linux, ACLs e permissões SMB, seguida de testes com diferentes identidades.
3. **Planejamento complementar:** avaliar a inclusão de uma máquina virtual Windows como cliente administrativo para explorar o gerenciamento de ACLs SMB por interface gráfica. Essa possibilidade ainda não faz parte da implementação concluída.
4. **Etapa 05 — Segurança:** hardening, auditoria e testes controlados.
5. **Etapa 06 — Consolidação:** testes integrados, diagrama e documentação geral do laboratório.

---

## Observações de documentação

Este documento registra o estado validado ao final da Etapa 02. As ocorrências intermediárias de rede, SMB, Kerberos, Chrony, AppArmor e serviços Samba foram resumidas para preservar os pontos relevantes sem transformar a documentação principal em uma transcrição integral do troubleshooting.

Os testes foram realizados no ambiente virtualizado descrito. Os resultados não devem ser interpretados como uma política de permissões finalizada ou como validação de acesso a partir de estações clientes externas ao próprio servidor.
