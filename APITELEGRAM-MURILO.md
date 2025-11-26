## API do Telegram

### Objetivo
A integração com o Telegram transforma conversas em tickets dentro do NexHelp. O backend expõe um **webhook** que recebe mensagens enviadas ao bot, cria/atualiza tickets e despacha respostas (humanas ou da IA) de volta ao usuário final.

### Arquitetura
1. **Webhook ASP.NET Core** (`Program.cs`):
   - Recebe o JSON do Telegram.
   - Valida e identifica o chat (`chat.id`).
   - Usa o caso de uso `CreateOrAppendTicket` para registrar a mensagem.
2. **Serviço `ITelegramNotifier`**:
   - Faz o envio de mensagens (texto, imagens, confirmações).
   - Armazena possíveis erros de entrega (`DeliveryError`) para reenvio manual.
3. **Repositório / Contexto**:
   - `TelegramConnection` guarda tokens, status e últimas atualizações.
   - `TicketMessage` persiste cada mensagem (cliente x operador/IA).

### Tecnologias
- **Telegram Bot API** via requisições HTTP.
- **HttpClient** configurado com timeouts curtos para evitar filas.
- **Entity Framework Core** para salvar cada interação antes de enviar respostas.

### Integração com os outros módulos
| Módulo | Papel na integração |
| --- | --- |
| Backend | Lê o webhook, decide se deve responder com IA ou operador, atualiza tickets. |
| Banco de Dados | Persiste mensagens e estado da conexão (token, fila, usuário proprietário). |
| OpenAI | Opcional: gera respostas somente se o ticket estiver com IA ativada. |
| Frontends | Exibem em tempo real as mensagens que chegaram via Telegram. |

### Fluxo simplificado
1. Cliente envia mensagem para o bot no Telegram.
2. Telegram chama o webhook (`/bot/update`) com os dados do chat/mensagem.
3. Backend grava a mensagem em `TicketMessage`, atualiza o ticket e, se estiver com IA ligada, dispara o processamento para o OpenAI.
4. Operadores visualizam e respondem pelo frontend; o backend envia a resposta ao Telegram através de `ITelegramNotifier`.

### Perguntas frequentes
| Pergunta | Resposta |
| --- | --- |
| O que acontece se o Telegram estiver offline? | A mensagem é salva e o campo `DeliveryError` fica preenchido. O operador consegue reenviar. |
| Como trocar o bot? | Atualize `TelegramConnection` com o novo token e reinicie o processo de webhook. |
| É possível desligar a IA para um ticket específico? | Sim. O toggle “Ativar/Desativar IA” altera o campo `UseAi`, e o backend respeita esse estado antes de chamar o OpenAI. |

### Exemplo de endpoint
```csharp
app.MapPost(webhookPath, async (HttpRequest req, IAiService aiService, ITelegramNotifier telegramNotifier, ...) =>
{
    var update = ParseTelegramPayload(req);
    var ticket = await useCase.HandleAsync(update.ChatId, update.Text);

    if (ticket.UseAi)
    {
        _ = Task.Run(async () =>
        {
            var response = await aiService.GenerateResponseAsync(...);
            await telegramNotifier.SendTextAsync(ticket.CustomerChatId, response);
        });
    }

    return Results.Ok();
});
```
O código acima ilustra:
- Recebimento assíncrono do update.
- Persistência imediata.
- Resposta da IA em `Task.Run` para não bloquear o webhook.

