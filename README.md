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



