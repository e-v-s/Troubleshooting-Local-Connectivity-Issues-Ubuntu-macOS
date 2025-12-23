# Troubleshooting: Local Connectivity Issues (Ubuntu ↔ macOS)

## Navigation

- [Troubleshooting: Local Connectivity Issues (Ubuntu ↔ macOS)](#troubleshooting-local-connectivity-issues-ubuntu--macos)
  - [Navigation](#navigation)
  - [Versions](#versions)
  - [Tecnologias Utilizadas](#tecnologias-utilizadas)
  - [Skills Utilizadas](#skills-utilizadas)
  - [Contexto](#contexto)
  - [Sintomas Observados](#sintomas-observados)
  - [Etapa 1: Diagnóstico de IPv4 vs IPv6](#etapa-1-diagnóstico-de-ipv4-vs-ipv6)
    - [Observação](#observação)
    - [Conclusão](#conclusão)
    - [Correção](#correção)
  - [Etapa 2: Verificação de Firewall](#etapa-2-verificação-de-firewall)
    - [Ubuntu](#ubuntu)
    - [macOS](#macos)
  - [Etapa 3: Verificação do Servidor SSH no Ubuntu](#etapa-3-verificação-do-servidor-ssh-no-ubuntu)
    - [Sintomas](#sintomas)
    - [Conclusão](#conclusão-1)
    - [Correção](#correção-1)
  - [Etapa 4: Interferência do libvirt / iptables](#etapa-4-interferência-do-libvirt--iptables)
    - [Teste realizado](#teste-realizado)
    - [Conclusão](#conclusão-2)
  - [Etapa 5: Teste Cruzado de Rede](#etapa-5-teste-cruzado-de-rede)
  - [Root Cause](#root-cause)
    - [Comportamento observado](#comportamento-observado)
    - [Impacto direto](#impacto-direto)
    - [Evidência](#evidência)
  - [Soluções Práticas Adotadas](#soluções-práticas-adotadas)
  - [Lições Aprendidas](#lições-aprendidas)
  - [Conclusão](#conclusão-3)

## Versions

- **Português (atual)**
- [English Version](./README_EN.md)

## Tecnologias Utilizadas

![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?logo=ubuntu&logoColor=white) ![macOS](https://img.shields.io/badge/macOS-000000?logo=apple&logoColor=white) ![KVM](https://img.shields.io/badge/KVM-00599C?logo=linux&logoColor=white) ![libvirt](https://img.shields.io/badge/libvirt-5A2D82) ![OpenSSH](https://img.shields.io/badge/OpenSSH-000000?logo=openssh&logoColor=white) ![iptables](https://img.shields.io/badge/iptables-777777) ![Network Troubleshooting](https://img.shields.io/badge/Networking-Debugging-blue)

## Skills Utilizadas

- Network troubleshooting (L2/L3/L7)
- Diagnóstico IPv4 vs IPv6
- Análise de firewall local (ufw / iptables)
- Debug de serviços Linux (systemctl, ssh, iptables, ufw)
- Análise de comportamento de roteadores de ISP
- Testes cruzados de conectividade
- Metodologia de troubleshooting estruturado

## Contexto

Durante a configuração do meu homelab (Ubuntu como host com KVM/libvirt, macOS como workstation), enfrentei uma série de problemas ao tentar realizar comunicação local entre o notebook Ubuntu (Lenovo ThinkPad T480) e um MacBook na mesma rede Wi-Fi residencial.

Os sintomas iniciais indicavam falhas em serviços aparentemente simples (HTTP local, SSH, SCP), o que levou a uma investigação completa envolvendo sistema operacional, firewall, virtualização e rede.

Este documento descreve o processo real de troubleshooting, os testes realizados, os falsos positivos descartados e o root cause final.

## Sintomas Observados

- Acesso à internet funcionava normalmente em ambos os dispositivos
- `ping www.google.com` funcionava
- `ping 8.8.8.8` inicialmente falhava (IPv4)
- `python3 -m http.server` iniciava corretamente no Ubuntu
- A porta aparecia em `LISTEN` (`ss -lntp`)
- Conexão via browser do macOS para o Ubuntu falhava
- `curl` retornava `recv failure: connection reset by peer`
- `nc` (netcat) conseguia conectar
- SSH (`ssh user@ip`) não conectava
- SCP não funcionava inicialmente

## Etapa 1: Diagnóstico de IPv4 vs IPv6

### Observação

- `ping www.google.com` funcionava
- `ping 8.8.8.8` não funcionava
- `ip route` mostrava apenas a rede `virbr0`

### Conclusão

O Ubuntu estava operando apenas com IPv6 ativo na interface Wi-Fi, o que explicava:

- Navegação funcionando
- Falha em serviços locais IPv4

### Correção

- Forçar IPv4 via NetworkManager
- Reconectar à rede Wi-Fi
- Confirmar presença de rota default IPv4 com `ip route`

Após essa correção, `ping 8.8.8.8` passou a funcionar.

**O problema persistiu para HTTP/SSH.**

## Etapa 2: Verificação de Firewall

### Ubuntu

- `ufw status` → inactive
- Nenhuma regra explícita bloqueando portas

### macOS

- Firewall desativado
- Nenhum proxy configurado
- Sem VPN ativa

Firewall descartado como causa.

## Etapa 3: Verificação do Servidor SSH no Ubuntu

### Sintomas

- `ssh localhost` inicialmente retornava `connection refused`
- `ss -lntp | grep :22` não mostrava nenhuma porta escutando
- `journalctl -xeu ssh` não mostrava erros

### Conclusão

O `openssh-server` não estava ativo corretamente.

### Correção

- Reinstalação do `openssh-server`
- Inicialização manual do serviço `ssh` via `systemctl`
- Confirmação de `LISTEN` na porta 22

Após isso, `ssh -v localhost` passou a funcionar corretamente.


## Etapa 4: Interferência do libvirt / iptables

Mesmo com `ufw` desativado, o `libvirt` havia inserido regras automáticas no `iptables`.

### Teste realizado

- Flush completo do `iptables`
- Políticas padrão definidas como `ACCEPT`
- `ssh localhost` continuou funcionando
- SSH remoto ainda não funcionava

### Conclusão

O bloqueio não estava no `iptables` do Ubuntu.


## Etapa 5: Teste Cruzado de Rede

Conexão via hotspot do celular (Ubuntu e macOS na mesma rede móvel):

- SSH funcionou imediatamente
- HTTP local funcionou


## Root Cause

O problema não estava nos sistemas, mas sim no roteador Wi-Fi residencial **F6600R (ZTE fornecido pelo ISP)**.

Este roteador aplica isolamento entre clientes Wi-Fi (client isolation) de forma silenciosa, mesmo na SSID principal.

### Comportamento observado

- Wi-Fi ↔ Wi-Fi bloqueado
- Wi-Fi ↔ Gateway permitido
- Internet funciona normalmente
- Comunicação lateral entre dispositivos não é permitida
- Não existe opção visível na interface para desativar esse comportamento

### Impacto direto

- SSH bloqueado
- HTTP local bloqueado
- Apenas conexões curtas (ex.: `nc`) permitidas
- Nenhum erro explícito no sistema operacional

### Evidência

- Comunicação falha na Wi-Fi do roteador ZTE
- Comunicação funciona imediatamente via hotspot móvel
- Nenhuma alteração adicional de configuração necessária

## Soluções Práticas Adotadas

- Uso de hotspot do celular para transferências locais quando necessário
- Uso de SSH/SCP apenas em redes confiáveis
- Evitar dependência de Wi-Fi de operadora para ambientes de laboratório
- Planejamento de uso de roteador próprio para o homelab

## Lições Aprendidas

- Conectividade com a internet não implica conectividade local
- IPv6 pode mascarar problemas de IPv4
- Nem todo bloqueio é visível via firewall ou configurações do sistema
- Roteadores de operadora não são ambientes confiáveis para labs técnicos
- Testes cruzados de rede são fundamentais para identificar root cause real

## Conclusão

Após investigação completa envolvendo sistema operacional, firewall, virtualização, rede e testes cruzados, o problema foi identificado como isolamento de clientes Wi-Fi imposto pelo roteador ZTE.

Nenhum erro foi encontrado na configuração do Ubuntu ou do macOS.

Este troubleshooting reforça a importância de validar a camada de rede física e lógica antes de assumir falhas de software.

