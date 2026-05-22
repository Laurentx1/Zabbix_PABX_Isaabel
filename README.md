Como vocês usam Issabel, o melhor jeito é usando:

Zabbix
n8n
AMI do Asterisk (que roda dentro do Issabel)
Objetivo

Quando um alerta crítico acontecer no Zabbix:

o Zabbix envia webhook
o n8n recebe
o Issabel faz ligação automática para um ramal
toca mensagem de alerta.
Arquitetura
Zabbix
   ↓ Webhook
n8n
   ↓ HTTP/AMI
Issabel (Asterisk)
   ↓
Ramal SIP
PASSO 1 — Habilitar AMI no Issabel

Acesse o servidor Issabel:

nano /etc/asterisk/manager_custom.conf

Adicione:

[alertabot]
secret=SenhaForte123
deny=0.0.0.0/0.0.0.0
permit=SEU_IP_N8N/255.255.255.255
read=all
write=all
Reinicie o Asterisk
asterisk -rx "manager reload"

ou:

systemctl restart asterisk
PASSO 2 — Testar porta AMI

AMI usa:

porta 5038

Teste:

telnet IP_DO_ISSABEL 5038

Deve aparecer:

Asterisk Call Manager
PASSO 3 — Criar contexto de chamada

Edite:

nano /etc/asterisk/extensions_custom.conf

Adicione:

[alerta-zabbix]
exten => 999,1,Answer()
 same => n,Playback(custom/alerta)
 same => n,Hangup()
PASSO 4 — Criar áudio

Coloque áudio:

/var/lib/asterisk/sounds/custom/alerta.wav

Formato:

WAV
mono
8000hz
PASSO 5 — Testar ligação manual

No Issabel:

asterisk -rx "channel originate SIP/200 extension 999@alerta-zabbix"

Troque:

200 = ramal

Se funcionar:

o telefone toca
áudio executa.
PASSO 6 — Criar webhook no n8n
Workflow
Node 1 — Webhook

Recebe alerta do Zabbix.

Exemplo payload:

{
  "host": "FW-MATRIZ",
  "severity": "DISASTER",
  "problem": "Firewall Offline"
}
PASSO 7 — Node HTTP Request

No n8n:

método: POST
URL:
http://IP_ISSABEL:8088/rawman
Body
Action: Login
Username: alertabot
Secret: SenhaForte123

Action: Originate
Channel: SIP/200
Context: alerta-zabbix
Exten: 999
Priority: 1

Action: Logoff
PASSO 8 — Configurar webhook no Zabbix

No:

Alerts
Media Types
Webhook

ou:

Actions → Operations → webhook.

URL:

http://IP_N8N/webhook/zabbix-alert
PASSO 9 — Filtrar apenas DISASTER

No n8n:

IF severity == DISASTER

Então:

faz ligação
senão ignora.
RESULTADO FINAL

Quando cair:

firewall
link
VM
Proxmox
servidor

O ramal toca automaticamente.
