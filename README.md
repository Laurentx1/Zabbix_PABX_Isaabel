# Alertas automáticos com Zabbix + n8n + Issabel

Quando um evento crítico acontece no Zabbix, o sistema dispara uma ligação automática para um ramal via Issabel (Asterisk), reproduzindo uma mensagem de alerta de voz.

---

## Arquitetura

```
Zabbix ──(webhook)──► n8n ──(AMI/HTTP)──► Issabel (Asterisk) ──► Ramal SIP
```

---

## Pré-requisitos

| Componente | Função |
|---|---|
| **Zabbix** | Monitoramento e disparo de alertas |
| **n8n** | Orquestração do webhook e chamada HTTP |
| **Issabel / Asterisk** | Central telefônica — faz a ligação |
| **Ramal SIP** | Destino da chamada de alerta |

---

## Passo 1 — Habilitar AMI no Issabel

Acesse o servidor e edite o arquivo de configuração do manager:

```bash
nano /etc/asterisk/manager_custom.conf
```

Adicione o usuário de acesso:

```ini
[alertabot]
secret=SenhaForte123
deny=0.0.0.0/0.0.0.0
permit=SEU_IP_N8N/255.255.255.255
read=all
write=all
```

Recarregue o manager sem precisar reiniciar tudo:

```bash
asterisk -rx "manager reload"
```

Ou, se preferir reiniciar o serviço completo:

```bash
systemctl restart asterisk
```

---

## Passo 2 — Testar conectividade com o AMI

O AMI escuta na porta `5038`. Verifique se está acessível:

```bash
telnet IP_DO_ISSABEL 5038
```

Resposta esperada:

```
Asterisk Call Manager/...
```

Se não conectar, verifique o firewall do servidor Issabel.

---

## Passo 3 — Criar contexto de chamada no Asterisk

Edite o arquivo de extensões customizadas:

```bash
nano /etc/asterisk/extensions_custom.conf
```

Adicione o contexto de alerta:

```ini
[alerta-zabbix]
exten => 999,1,Answer()
 same => n,Playback(custom/alerta)
 same => n,Hangup()
```

> O Asterisk vai atender a chamada, reproduzir o arquivo de áudio `alerta` e desligar.

---

## Passo 4 — Criar o áudio de alerta

Coloque o arquivo de áudio no caminho correto:

```
/var/lib/asterisk/sounds/custom/alerta.wav
```

Especificações obrigatórias do arquivo:

- Formato: **WAV**
- Canais: **mono**
- Taxa de amostragem: **8000 Hz**

> Arquivos fora desse padrão podem não tocar corretamente ou causar ruído.

---

## Passo 5 — Testar a ligação manualmente

Antes de integrar com o n8n, valide que o Asterisk consegue fazer a ligação:

```bash
asterisk -rx "channel originate SIP/200 extension 999@alerta-zabbix"
```

Substitua `200` pelo número do ramal de destino.

**Resultado esperado:** o telefone do ramal toca e o áudio de alerta é reproduzido.

---

## Passo 6 — Criar workflow no n8n

### Node 1 — Webhook

Configure um node **Webhook** para receber os alertas do Zabbix.

Exemplo de payload enviado pelo Zabbix:

```json
{
  "host": "FW-MATRIZ",
  "severity": "DISASTER",
  "problem": "Firewall Offline"
}
```

---

## Passo 7 — Node HTTP Request (originar chamada)

Adicione um node **HTTP Request** com as seguintes configurações:

| Campo | Valor |
|---|---|
| Método | `POST` |
| URL | `http://IP_ISSABEL:8088/rawman` |

Body da requisição (texto plano, `Content-Type: text/plain`):

```
Action: Login
Username: alertabot
Secret: SenhaForte123

Action: Originate
Channel: SIP/200
Context: alerta-zabbix
Exten: 999
Priority: 1

Action: Logoff
```

> Substitua `SIP/200` pelo ramal que deve receber a ligação.

---

## Passo 8 — Configurar webhook no Zabbix

No Zabbix, vá em **Alerts → Media Types → Webhook** (ou **Actions → Operations**) e configure:

| Campo | Valor |
|---|---|
| URL | `http://IP_N8N/webhook/zabbix-alert` |
| Método | `POST` |
| Content-Type | `application/json` |

Vincule o media type a um usuário e configure o trigger de alerta para usar essa action.

---

## Passo 9 — Filtrar apenas alertas críticos

No n8n, adicione um node **IF** entre o Webhook e o HTTP Request:

```
severity == DISASTER
```

- **Verdadeiro** → executa o HTTP Request (faz a ligação)
- **Falso** → encerra o workflow sem ação

Isso evita ligações desnecessárias para alertas de baixa severidade.

---

## Resultado

Quando qualquer um dos eventos abaixo for detectado pelo Zabbix com severidade **DISASTER**:

- Firewall offline
- Link de internet caído
- VM inacessível
- Host Proxmox fora do ar
- Servidor sem resposta

→ O ramal configurado **toca automaticamente** com a mensagem de alerta de voz.

---

## Dicas adicionais

**Múltiplos ramais:** para ligar para mais de um ramal, repita o bloco `Action: Originate` no body com canais diferentes, ou crie um loop no n8n.

**Mensagens dinâmicas:** use o campo `problem` do payload do Zabbix para gerar áudios com text-to-speech (ex: via ElevenLabs ou Google TTS) antes de chamar o Issabel.

**Retry em caso de falha:** adicione um node de retry no n8n caso o Issabel não responda na primeira tentativa.
