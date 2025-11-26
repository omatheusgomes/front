## API do OpenAI

### Objetivo
Prover respostas automáticas em tickets que estiverem com a IA ativada. Sempre que o cliente envia uma mensagem e o operador não desativou a IA, o backend chama o OpenAI, formata a resposta e grava como `TicketMessage`.

### Arquitetura
1. **Serviço `IAiService`** (Application):
   - Define o contrato `GenerateResponseAsync`.
2. **`OpenAiService` (Infrastructure)**:
   - Implementa o contrato usando `HttpClient`.
   - Monta o payload com histórico e instruções.
   - Respeita limites de tokens, temperatura, etc.
3. **Repositório `IOpenAiPromptRepository`**:
   - Permite configurar prompts específicos por usuário ou fila.
   - Fica armazenado em `OpenAiPrompts` (tabela com `MaxTokens`, `Queues`, `IsActive`, chave do usuário).

### Tecnologias
- **OpenAI REST API** (modelo padrão configurado em `appsettings.json`, ex.: `gpt-4o-mini`).
- **HttpClient** com timeout de 10 segundos para evitar bloqueios.
- **JSON (System.Text.Json)** para serializar/de-serializar o payload.

### Integração com demais módulos
| Módulo | Integração |
| --- | --- |
| Backend | Decide se chama a IA (verifica `UseAi` e estado do toggle no ticket). |
| Banco de Dados | Guarda prompts, histórico das mensagens e se a IA está ativa para o ticket. |
| Telegram API | Se o ticket for do Telegram, a resposta da IA volta para o cliente via `ITelegramNotifier`. |
| Frontends | Exibem o toggle “Ativar/Desativar IA” e mostram a mensagem gerada como se fosse uma resposta comum. |

### Fluxo resumido
1. Cliente manda mensagem → backend cria `TicketMessage`.
2. `Ticket.UseAi == true`? Se sim:
   - Busca prompt ativo (do usuário ou padrão).
   - Monta o histórico (últimas mensagens).
   - Chama `OpenAiService`.
   - Salva a resposta como `TicketMessage` com `IsAi = true`.
   - Envia a resposta para o cliente (Telegram) e atualiza os frontends.
3. Se o operador desativar a IA no toggle, o backend simplesmente ignora esse passo.

### Perguntas frequentes
| Pergunta | Resposta |
| --- | --- |
| Onde fica a chave da OpenAI? | `appsettings.json` (chave `AI:ApiKey`) ou no prompt específico salvo no banco. |
| Posso limitar tokens ou histórico? | Sim, cada prompt tem `MaxTokens` e `MaxHistory`. |
| Como garantir respostas apenas técnicas? | O prompt padrão inclui instruções “Responda apenas suporte técnico. Caso contrário, informe que não tratamos desse assunto”. |

### Exemplo simplificado de chamada
```csharp
var payload = new {
    model = options.Model,
    temperature = 0.5,
    max_tokens = prompt.MaxTokens ?? 150,
    messages = BuildHistory(ticket.Messages)
};

var httpResponse = await _httpClient.PostAsync("https://api.openai.com/v1/chat/completions",
    new StringContent(JsonSerializer.Serialize(payload), Encoding.UTF8, "application/json"));
```
Esse serviço retorna apenas o texto relevante, que é salvo e apresentado exatamente como se o operador tivesse digitado.

