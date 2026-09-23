# Etapa 01 — Controlador de Domínio Samba AD/DC

## Objetivo

Implantar o primeiro serviço central da infraestrutura: um Controlador de Domínio Active Directory utilizando Samba no Ubuntu Server.

A etapa estabelece a base de identidade e autenticação do laboratório, utilizando Samba, DNS interno e Kerberos.

## Ambiente

- Sistema operacional: Ubuntu Server 22.04.5 LTS
- Hostname: `ubuntu-dc`
- FQDN: `ubuntu-dc.lab.internal`
- Domínio: `lab.internal`
- Realm Kerberos: `LAB.INTERNAL`
- NetBIOS: `LAB`
- IP estático: `192.168.10.10`
- Rede: `192.168.10.0/24`
- Interface: `enp0s3`

## 1. Configuração de rede

O laboratório utiliza uma rede interna isolada para a comunicação entre as máquinas virtuais.

Quando foi necessário instalar ou atualizar pacotes, a máquina virtual foi temporariamente conectada a uma rede em modo NAT para acesso aos repositórios. Após a manutenção, a conexão foi restaurada para a rede interna do laboratório.

> **Nota de documentação:** As anotações iniciais continham configurações diferentes de renderizador, rota padrão e DNS. Durante a evolução do laboratório, esses parâmetros foram ajustados. Esta documentação registra o estado atualmente validado no ambiente.

### Configuração de rede validada

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

### Validação

A configuração ativa da interface pode ser verificada com:

```bash
nmcli connection show
```

A conexão `netplan-enp0s3` deve aparecer associada à interface `enp0s3`.

A configuração de resolução de nomes pode ser verificada com:

```bash
cat /etc/resolv.conf
```

Resultado esperado:

```text
nameserver 192.168.10.10
search lab.internal
domain lab.internal
```

Essa configuração permite que o próprio controlador de domínio seja utilizado como servidor DNS para o ambiente interno do laboratório.
