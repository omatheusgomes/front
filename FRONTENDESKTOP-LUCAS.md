## Frontend Desktop

### Objetivo
Disponibilizar um aplicativo desktop (Tkinter + Python) para operadores que preferem uma janela dedicada. Ele utiliza os mesmos endpoints da API e espelha funcionalidades do frontend web: login, visualização de chamados, contatos, usuários, dashboards e chat.

### Arquitetura
1. **Tkinter** organiza as telas em `screens/*.py` (ex.: `chamados_screen.py`, `usuarios_screen.py`).
2. **Camada de serviços** (`services/api_client.py`, `auth.py`) centraliza chamadas HTTP e controle de sessão.
3. **Gestão de estado** simples com arquivos JSON (`.nexhelp_desktop_session.json`) para lembrar o token e preferências.

### Tecnologias
- **Python 3 + Tkinter** para a interface.
- **Requests** (ou `httpx`) para acessar a API.
- **PIL (Pillow)** para manipulação de ícones/imagens no painel lateral.

### Integração com o backend e demais módulos
| Funcionalidade | Integração |
| --- | --- |
| Login | `POST /api/auth/login` (mesma rota do web). |
| Atualização de tickets | Polling periódico chamando `GET /api/tickets`. |
| Chat | `GET/POST /api/tickets/{id}/messages` e, quando necessário, `POST /api/telegram/ticket/{id}/send`. |
| IA | O desktop respeita o `UseAi` vindo da API. Se o operador desliga a IA em qualquer frontend, o desktop se ajusta automaticamente. |

### Experiência do usuário
- **Listas com fontes maiores** (Courier New + espaçamento) para facilitar leitura.
- **Ícones vetoriais gerados em tempo real** (`icon_loader.py`) usando emoji fonts e ajustes finos para centralizar cada símbolo na sidebar.
- **Controles de acesso**: técnicos conseguem visualizar módulos, mas só administradores podem gerenciar usuários/conexões (mesma regra do backend).

### Perguntas frequentes
| Pergunta | Resposta |
| --- | --- |
| O desktop funciona offline? | Não. Ele depende da mesma API, mas mantém o token salvo para facilitar reconexão. |
| Como o layout fica responsivo? | Tkinter usa `grid`/`pack` com pesos configurados; fontes e espaçamentos foram ajustados manualmente para cada tela. |
| O chat do desktop usa IA? | Sim. Ele apenas reflete o estado do ticket. Se a IA estiver ativa, as respostas automáticas chegam e são renderizadas no listbox. |

### Exemplo simples de chamada
```python
from services.api_client import ApiClient

api = ApiClient(base_url, token)
tickets = api.get_tickets()
for t in tickets:
    chamados_listbox.insert('end', f"#{t['sequentialId']} - {t['title']}")
```
Esse trecho ilustra como cada tela se mantém sincronizada com o backend reutilizando o mesmo cliente HTTP.

