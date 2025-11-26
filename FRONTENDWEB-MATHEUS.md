## Frontend Web

### Objetivo
Oferecer uma interface responsiva (desktop e mobile) para operadores acompanharem tickets, responder clientes, configurar integrações e visualizar indicadores. Todo o front roda em HTML, CSS e JavaScript puro (sem frameworks pesados), consumindo a API via `fetch`.

### Arquitetura
1. **Páginas HTML** (ex.: `index.html`, `chamados.html`, `usuarios.html`).
2. **CSS modularizado** (`layout.css`, `portrait.css`, `landscape.css`) para adaptar a mesma tela a diferentes dispositivos.
3. **Módulos JS** (`frontend-web/js/*.js`) responsáveis por:
   - Guardar sessões (`services/auth.js`).
   - Chamar a API (`services/api.js`).
   - Controlar cada tela (`chamados.js`, `usuarios.js`, etc.).

### Tecnologias
- HTML5 + CSS3 (com media queries e grid layouts).
- JavaScript ES modules + `fetch`.
- LocalStorage para guardar o token JWT.
- Ícones SVG locais (carregados dinamicamente).

### Integração com o backend
| Ação | Endpoint | Observação |
| --- | --- | --- |
| Login | `POST /api/auth/login` | Token armazenado em localStorage. |
| Lista de tickets | `GET /api/tickets` | Atualiza a cada poucos segundos para manter a caixa de entrada. |
| Responder cliente | `POST /api/tickets/{id}/messages` ou `POST /api/telegram/ticket/{id}/send` | O JS escolhe automaticamente dependendo da origem. |
| Toggle IA | `PUT /api/tickets/{id}/use-ai` | O estado do switch fica sincronizado com o backend. |

### Pontos de atenção
- **Chamados no mobile**: há uma versão “portrait” que remove a sidebar e prioriza apenas a lista + chat (pensando em futuramente virar APK via WebView).
- **IA e desempenho**: o script “chamados.js” otimiza o carregamento de mensagens com logs de performance, evita duplicação e respeita o estado do toggle.
- **Segurança**: antes de carregar qualquer página protegida, `guards.js` verifica se o token ainda é válido; caso contrário, redireciona para o login.

### Exemplo simples de consumo de API
```javascript
const tickets = await api.getTickets();
tickets.forEach(t => renderTicket(t));
```
Esse padrão se repete em cada módulo: uma função na camada `services/api.js` faz a chamada autenticada e o controlador da tela renderiza o resultado.

### Perguntas frequentes
| Pergunta | Resposta |
| --- | --- |
| Como o front carregado no mobile se adapta? | `portrait.css` redefine o grid para esconder a sidebar e empilhar lista + chat. |
| Existe cache offline? | Não. A prioridade foi manter o estado atualizado em tempo real via polling leve. |
| É possível desligar a IA diretamente no web? | Sim. O switch na topbar já sincroniza com o backend e altera o comportamento da IA naquele ticket. |

