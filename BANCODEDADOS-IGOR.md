## Banco de Dados

### Como o banco é criado
- O NexHelp usa **Entity Framework Core** e **migrations** para criar e atualizar o esquema.
- Não existe um script `.sql` com `CREATE DATABASE`. Em vez disso, rodamos o comando:
  ```
  dotnet ef database update
  ```
  Esse comando:
  1. Conecta no SQL Server usando a connection string configurada.
  2. Cria o banco automaticamente (se não existir).
  3. Aplica todas as migrations, criando tabelas, índices e relacionamentos.
- Connection string (exemplo em `appsettings.json`):
  ```json
  "ConnectionStrings": {
    "Default": "Server=(localdb)\\MSSQLLocalDB;Database=nexhelp;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=True"
  }
  ```
  Ela aponta para o SQL Server local (LocalDB). Em produção, basta trocar o valor para o servidor desejado.

### Arquitetura e tecnologias
- **SQL Server** como banco relacional.
- **Entity Framework Core** como ORM (mapeia classes para tabelas).
- **ASP.NET Identity** adiciona tabelas extras para usuários, perfis e tokens.

### Estrutura principal
| Entidade | Função | Tabelas/Relacionamentos |
| --- | --- | --- |
| `Ticket` | Representa cada chamado | Relaciona com `TicketMessage`, `TicketTag` e `ApplicationUser` |
| `TicketMessage` | Mensagens trocadas com o cliente | FK para `Ticket` |
| `Tag` e `TicketTag` | Classificação flexível | Relacionamento muitos-para-muitos |
| `TelegramConnection` | Armazena tokens/estado de bots | Permite múltiplas conexões por usuário |
| `OpenAiPrompt` | Configurações por fila/usuário | Define parâmetros para chamadas à IA |

### Integração com outras partes
- **Backend**: o `AppDbContext` injeta as DbSets e EF gera as queries.
- **Telegram API**: mensagens recebidas são persistidas em `TicketMessage` antes de qualquer resposta.
- **OpenAI**: quando a IA responde, o texto vira um `TicketMessage` com a flag `IsAi`.
- **Frontends**: consomem endpoints expostos pelo backend que, por sua vez, leem/escrevem no banco via EF.

### Perguntas comuns
| Pergunta | Resposta |
| --- | --- |
| Como recriar o banco? | `dotnet ef database update` em `NexHelp.Infrastructure`, especificando o projeto API como startup. |
| Como versionar mudanças futuras? | `dotnet ef migrations add NomeDaMudanca` + `dotnet ef database update`. |
| É possível usar outro servidor SQL? | Sim. Basta ajustar a connection string e garantir que o usuário tenha permissão para criar/aplicar migrations. |

### Exemplo simples de migration
```csharp
public partial class AddUseAiToTickets : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<bool>(
            name: "UseAi",
            table: "Tickets",
            type: "bit",
            nullable: false,
            defaultValue: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "UseAi",
            table: "Tickets");
    }
}
```
Esse padrão garante que toda alteração possa ser aplicada ou revertida de maneira controlada.

