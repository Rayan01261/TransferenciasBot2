# GRP Automação de Bens Móveis

Servidor **Flask + Socket.IO** que automatiza, via **Selenium**, tarefas repetitivas no sistema GRP (`tjse.thema.inf.br`): consulta de bens móveis, transferências de patrimônio (individuais e em massa) e download dos termos de transferência em PDF, já renomeados com o número do chamado e da GRP.

O front-end (navegador) se comunica com o backend por WebSocket, acompanha o progresso em tempo real, informa o código de 2FA quando solicitado e baixa os PDFs gerados.

---

## Funcionalidades

- **Consulta de bem** individual e **em massa** (resultado salvo em `resultado.txt`)
- **Transferência de bem** com rota fixa em duas etapas (origem → `4128` → `4126`)
- **Transferência para destino informado** (individual)
- **Transferência em massa** a partir de `resultado.txt` ou de uma lista enviada pelo front
- **Fila de tarefas** processada por um worker, com status em tempo real (`aguardando`, `executando`, `concluida`, `erro`)
- **Login a cada ação**: cada tarefa abre um Chrome novo (headless), faz login e fecha ao final, sem reaproveitar sessão
- **Suporte a 2FA**: o backend pausa e pede o código de 6 dígitos ao usuário via front
- **Extração do PDF** da GRP: lê o número da GRP no PDF, renomeia para `<chamado>_<grp>.pdf` e envia ao navegador em base64
- Logs de progresso enviados apenas ao cliente que disparou a ação

---

## Arquitetura

```
Navegador (index.html)
        │  Socket.IO
        ▼
Flask-SocketIO ──► Fila (queue.Queue) ──► worker()
                                             │
                                             ▼
                              executar_com_login()
                          ┌───────────┴────────────┐
                          ▼                        ▼
                  abrir() + 2FA            ação de negócio
                  (login no GRP)     (consulta / transferir / ...)
                                                   │
                                                   ▼
                                  PDF → renomear → emit_pdf()
```

Principais pontos do código:

| Função | Papel |
|---|---|
| `registrar_tarefa_na_fila()` | Cria o status da tarefa, faz broadcast e enfileira |
| `worker()` | Consome a fila e executa uma tarefa por vez |
| `executar_com_login()` | Driver novo + login + ação + `driver.quit()` |
| `verificar_e_tratar_2fa()` | Detecta 2FA, pede o código ao front e preenche |
| `_extrair_pdf_transferencia()` | Gera o relatório em PDF e aciona o download |
| `renomear_e_mover_pdf*()` | Lê o PDF, extrai o nº da GRP, renomeia e entrega |

---

## Requisitos

- Python 3.9+
- Google Chrome + ChromeDriver compatível (ou Selenium Manager)
- Acesso de rede ao GRP (e ao proxy, se houver)

Dependências Python:

```bash
pip install selenium flask flask-socketio PyPDF2
```

---

## Instalação e execução

```bash
git clone https://github.com/<seu-usuario>/<seu-repo>.git
cd <seu-repo>

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python app.py
```

O servidor sobe em `http://0.0.0.0:5100`. Acesse `http://localhost:5100` no navegador.

> A rota `/` renderiza `templates/index.html`, então mantenha esse arquivo na pasta `templates/`.

---

## Configuração

No topo do script:

| Variável | Descrição |
|---|---|
| `HTTPS_PROXY` | Proxy da rede. Ajuste para o endereço real, **sem** `/wpad.dat` |
| `ARQUIVO_ENTRADA` | Arquivo de entrada de patrimônios (`patrimonios.txt`) |
| `ARQUIVO_SAIDA` | Arquivo de resultado das consultas (`resultado.txt`) |
| `LOGIN_USUARIO` / `LOGIN_SENHA` | Valores *fallback*. Em uso normal, o usuário e a senha vêm do front em cada ação |
| `SECRET_KEY` | Troque o valor padrão antes de qualquer uso fora de testes |

> ⚠️ **Nunca** versione credenciais reais. Mantenha `LOGIN`/`SENHA` como placeholders (ou use variáveis de ambiente / `.env` no `.gitignore`).

---

## Eventos Socket.IO

### Cliente → Servidor

Todos os eventos de ação exigem `usuario` e `senha` no payload.

| Evento | Payload | Descrição |
|---|---|---|
| `consulta` | `{objeto, usuario, senha}` | Consulta um bem |
| `consulta_em_massa` | `{objetos: [...], usuario, senha}` | Consulta vários bens e grava `resultado.txt` |
| `transferir` | `{objeto, chamado, usuario, senha}` | Transferência em duas etapas (rota fixa) |
| `transferir_recebimento` | `{objeto, destino, chamado, usuario, senha}` | Transferência de um bem ao destino informado |
| `transferir_em_massa` | `{destino, chamado, usuario, senha}` | Transfere os bens listados em `resultado.txt` |
| `transferir_recebimento_em_massa` | `{objetos: [...], origem, destino, chamado, usuario, senha}` | Transferência em massa com lista vinda do front |
| `enviar_2fa` | `{codigo}` | Envia o código de 6 dígitos solicitado pelo servidor |

### Servidor → Cliente

| Evento | Descrição |
|---|---|
| `log` | `{tipo, mensagem}`: progresso (`info`, `alerta`, `erro`) |
| `fila_atualizada` | `{tarefas: [...]}`: status atual da fila (broadcast) |
| `solicitar_2fa` | O front deve pedir o código de 2FA ao usuário |
| `pdf_disponivel` | `{nome_arquivo, conteudo_base64}`: PDF pronto para download |
| `resultado_<acao>` | `{status, resultado}` ou `{status: "erro", mensagem}` ao final da ação |

---

## Formato dos arquivos

`resultado.txt` (gerado por `consulta_em_massa`, lido por `transferir_em_massa`):

```
<patrimonio> | <codigo_local> - <descricao>
<patrimonio> | inexistente
```

Patrimônios que não puderem ser inseridos na transferência em massa são anexados em `Patrimonios_ruins.txt`.

---

## Observações importantes

- **Fila sequencial**: apenas um worker processa as tarefas, uma por vez. O fluxo depende da pasta `Downloads` do usuário que roda o servidor.
- **Pasta Downloads**: o PDF mais recente de `~/Downloads` é lido, enviado ao navegador e **removido** da pasta. Evite usar a máquina para outros downloads de PDF durante a execução.
- **Rotas fixas**: `transferir` usa os locais `4128` e `4126` fixos no código. Ajuste conforme a sua realidade.
- **Seletores frágeis**: os XPaths/IDs dependem do HTML do GRP. Mudanças no sistema podem exigir ajustes.
- **Dados sensíveis**: usuário e senha trafegam pelo WebSocket. Use HTTPS/WSS (ex.: atrás de um proxy reverso) e restrinja `cors_allowed_origins` em produção.

---

## Estrutura sugerida do repositório

```
.
├── app.py
├── templates/
│   └── index.html
├── requirements.txt
├── .gitignore
└── README.md
```

Sugestão de `.gitignore`:

```
.venv/
__pycache__/
resultado.txt
patrimonios.txt
Patrimonios_ruins.txt
.env
```

---

## Licença

Defina a licença do projeto (ex.: MIT) e adicione o arquivo `LICENSE`.
