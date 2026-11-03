# Etapa 01 — Controlador de Domínio Samba AD/DC

## Objetivo

Implantar o primeiro serviço central da infraestrutura: um Controlador de Domínio Active Directory utilizando Samba no Ubuntu Server.

Esta etapa estabelece a base de identidade, autenticação e resolução de nomes do laboratório, preparando o ambiente para a integração de outros servidores ao domínio.

## Ambiente

- **Sistema operacional:** Ubuntu Server 22.04.5 LTS
- **Hostname:** `ubuntu-dc`
- **FQDN:** `ubuntu-dc.lab.internal`
- **Domínio:** `lab.internal`
- **Realm Kerberos:** `LAB.INTERNAL`
- **NetBIOS:** `LAB`
- **IP estático:** `192.168.10.10`
- **Rede:** `192.168.10.0/24`	
- **Interface de rede:** `enp0s3`

## 1. Configuração de rede

O laboratório utiliza uma rede interna isolada para permitir a comunicação entre as máquinas virtuais.

Durante a instalação e manutenção dos sistemas, foi necessário utilizar temporariamente uma conexão em modo NAT para acessar os repositórios de pacotes. Após essas operações, a conexão foi restaurada para a rede interna do laboratório.

### Configuração de rede

A interface `enp0s3` foi configurada com endereço IP estático, utilizando o NetworkManager como renderizador.

Arquivo de configuração: `/etc/netplan/01-network-manager-all.yaml`

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.10.10/24
      nameservers:
        addresses:
          - 192.168.10.10
        search:
          - lab.internal
```

### Validação

A conexão ativa pode ser verificada com:

```bash
nmcli connection show
```

A conexão `netplan-enp0s3` deve aparecer associada à interface `enp0s3`.

A configuração de resolução de nomes pode ser consultada com:

```bash
cat /etc/resolv.conf
```

Configuração esperada:

```text
nameserver 192.168.10.10
search lab.internal
domain lab.internal
```

O endereço do próprio controlador de domínio é utilizado como servidor DNS para a resolução de nomes dentro do laboratório.

### Observações

As configurações iniciais de rede passaram por revisões durante a implantação. Este documento registra a configuração validada ao final dessa etapa, preservando as alterações relevantes para a compreensão do ambiente.

## 2. Serviços de domínio

O Ubuntu Server foi utilizado para implantar o Samba como Controlador de Domínio Active Directory (AD/DC).

**Parâmetros do domínio:**

| Parâmetro | Valor |
|---|---|
| Nome do domínio | `lab.internal` |
| Realm Kerberos | `LAB.INTERNAL` |
| Nome NetBIOS | `LAB` |
| Controlador de domínio | `ubuntu-dc` |
| Endereço IP | `192.168.10.10` |

## 3. Estado da etapa

Ao final desta etapa, o Ubuntu Server passou a atuar como base central de domínio para a infraestrutura do laboratório.

O controlador de domínio foi utilizado posteriormente como referência para a integração do servidor Debian, desenvolvido na Etapa 02.
