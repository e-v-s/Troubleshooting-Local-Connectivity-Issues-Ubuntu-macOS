# Troubleshooting: Local Connectivity Issues (Ubuntu ↔ macOS)

## Contexto

Durante a configuração do meu homelab (Ubuntu como host com KVM/libvirt, macOS como workstation), enfrentei uma série de problemas ao tentar realizar comunicação local entre o notebook Ubuntu (Lenovo ThinkPad T480) e um MacBook na mesma rede Wi-Fi residencial.

Os sintomas iniciais indicavam falhas em serviços aparentemente simples (HTTP local, SSH, SCP), o que levou a uma investigação completa envolvendo sistema operacional, firewall, virtualização e rede.

Este documento descreve o processo real de troubleshooting, os testes realizados, os falsos positivos descartados e o root cause final.

## Sintomas Observados

- Acesso à internet funcionava normalmente em ambos os dispositivos
- ping www.google.com funcionava
- ping 8.8.8.8 inicialmente falhava (IPv4)
- python3 -m http.server iniciava corretamente no Ubuntu
- A porta aparecia em LISTEN (ss -lntp)
- Conexão via browser do macOS para o Ubuntu falhava
- curl retornava recv failure: connection reset by peer
- nc (netcat) conseguia conectar
- SSH (ssh user@ip) não conectava
- SCP não funcionava inicialmente

## Etapa 1: Diagnóstico de IPv4 vs IPv6

### Observação

- ping www.google.com funcionava
- ping 8.8.8.8 não funcionava
- ip route mostrava apenas a rede virbr0

### Conclusão

O Ubuntu estava operando apenas com IPv6 ativo na interface Wi-Fi, o que explicava:

- Navegação funcionando
- Falha em serviços locais IPv4

### Correção

- Forçar IPv4 via NetworkManager
- Reconectar à rede Wi-Fi
- Confirmar presença de rota default IPv4 com ip route

Após essa correção `ping 8.8.8.8` passou a funcionar.

**O problema persistiu para HTTP/SSH**

## Etapa 2: Verificação de Firewall

### Ubuntu

- ufw status → inactive
- Nenhuma regra explícita bloqueando portas

### macOS

- Firewall desativado
- Nenhum proxy configurado
- Sem VPN ativa

Firewall descartado como causa.

## Etapa 3: Verificação do Servidor SSH no Ubuntu

### Sintomas

- ssh localhost inicialmente retornava connection refused
- ss -lntp | grep :22 não mostrava nenhuma porta escutando
- journalctl -xeu ssh não mostrava erros

### Conclusão

O openssh-server não estava ativo corretamente.

### Correção

- Reinstalação do openssh-server
- Inicialização manual do serviço ssh via systemctl
- Confirmação de LISTEN na porta 22

### Após isso

`ssh -v localhost`passou a funcionar corretamente

## Etapa 4: Interferência do libvirt / iptables

Mesmo com ufw desativado, o libvirt havia inserido regras automáticas no iptables.

### Teste realizado

- Flush completo do iptables
- Políticas padrão definidas como ACCEPT
- `ssh -v localhost` continuou funcionando
- SSH remoto ainda não funcionava

### Conclusão
O bloqueio não estava no iptables do Ubuntu.

## Etapa 5: Teste Cruzado de Rede

Conexão via hotspot do celular (Ubuntu e macOS na mesma rede móvel):

- SSH funcionou imediatamente
- HTTP local funcionou

## Conclusão Final

O problema não estava nos sistemas, mas sim no roteador Wi-Fi residencial F6600R (ZTE fornecido pelo ISP). Este é um problema conhecido, inclusive.

## Root Cause

O roteador ZTE aplica isolamento entre clientes Wi-Fi (client isolation):

- Wi-Fi ↔ Wi-Fi bloqueado
- Wi-Fi ↔ Gateway permitido
- Internet funciona normalmente
- Comunicação lateral entre dispositivos não é permitida
- Não existe opção visível na interface para desativar esse comportamento

### Esse isolamento

- Bloqueava SSH
- Bloqueava HTTP local
- Permitia apenas conexões curtas (ex.: nc)
- Não apresentava erros claros no sistema operacional

### Evidência

- Comunicação falha na Wi-Fi do roteador ZTE
- Comunicação funciona imediatamente via hotspot móvel
- Nenhuma alteração adicional de configuração necessária

### Soluções Práticas Adotadas

- Usar hotspot do celular para transferência de arquivos quando necessário
- Utilizar em redes confiáveis
- Evitar depender de Wi-Fi de operadora para ambientes de laboratório

## Lições Aprendidas

- Conectividade com a internet não implica conectividade local
- IPv6 pode mascarar problemas de IPv4
- Nem todo bloqueio é visível via firewall ou configurações do OS
- Roteadores de operadora não são ambientes confiáveis para labs técnicos

## Conclusão

Após investigação completa envolvendo sistema operacional, firewall, virtualização, rede e testes cruzados, o problema foi identificado como isolamento de clientes Wi-Fi imposto pelo roteador ZTE.

Nenhum erro foi encontrado na configuração do Ubuntu ou do macOS.

Este troubleshooting reforça a importância de sempre validar a camada de rede física e lógica antes de assumir falhas de software.