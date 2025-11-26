## Backend

### Visão Geral
O backend do NexHelp é um conjunto de serviços escritos em **C# (.NET 8)** que centralizam autenticação, gerenciamento de usuários, tickets e integrações externas. Ele segue uma arquitetura em camadas (Domain → Application → Infrastructure → Api), o que separa regras de negócio das operações de infraestrutura e facilita testes e evolução.

### Arquitetura e Princípios de Orientação a Objetos
- **Classes e responsabilidade única**: `Ticket`, `TicketMessage`, `TelegramConnection` e outras classes do domínio representam entidades com propriedades e comportamentos específicos.
- **Herança**: `AppDbContext` herda de `IdentityDbContext<ApplicationUser>`, aproveitando toda a infraestrutura do ASP.NET Identity.
- **Polimorfismo / Inversão de dependência**: interfaces como `ITicketRepository` e `IOpenAiPromptRepository` são implementadas no projeto Infrastructure. A API consome essas interfaces sem saber o detalhe da implementação concreta.

```csharp
public class Ticket
{
    public Guid Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public ICollection<TicketMessage> Messages { get; set; } = new List<TicketMessage>();
}

public interface ITicketRepository
{
    Task<IEnumerable<Ticket>> GetAllAsync(string? userId, TicketStatus? status);
    Task AddMessageAsync(TicketMessage message);
}
```
O código acima mostra:
1. **Classe** `Ticket` com propriedades e relação com `TicketMessage`.
2. **Interface** `ITicketRepository` que permite múltiplas implementações (polimorfismo) sem mudar o restante do sistema.

### Tecnologias
- **.NET 8 / ASP.NET Core Minimal APIs**
- **Entity Framework Core** para acesso ao banco de dados SQL Server
- **ASP.NET Identity** para autenticação e autorização
- **HttpClient** e serviços específicos para OpenAI e Telegram

### Integração com outros módulos
1. **Banco de dados**: o backend é responsável por aplicar as migrations e persistir dados de tickets, mensagens, usuários e integrações.
2. **API do Telegram**: expõe endpoints (webhook e utilitários) que recebem mensagens do Telegram, criam ou atualizam tickets e disparam notificações.
3. **API do OpenAI**: serviços de IA são chamados a partir dos casos de uso do backend sempre que o ticket estiver com a IA habilitada.
4. **Frontends (Web e Desktop)**: ambos consomem os endpoints expostos pelo backend para autenticar, listar dados e acionar comandos (transferir, finalizar, responder chamados etc.).

### Pontos que costumam gerar perguntas
| Tema | Resposta curta |
| --- | --- |
| Como o backend garante segurança? | JWT + ASP.NET Identity controlam autenticação e autorizam por perfil (Admin/Técnico). |
| Como é mantida a consistência? | Camadas separadas + repositórios garantem que regras de negócio passem sempre pelos mesmos fluxos. |
| O que acontece se o Telegram falhar? | O backend salva a mensagem primeiro e registra o erro de entrega; o operador pode reenviar. |

### Exemplo de fluxo
1. Frontend envia `POST /api/tickets/{id}/messages`.
2. Backend valida o usuário, salva a mensagem via `ITicketRepository`, atualiza o ticket e, se necessário, chama o Telegram e a IA.
3. Frontend recebe a resposta e atualiza a interface sem sair da tela.

