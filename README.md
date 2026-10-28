# PredictMaint · Mantech Solutions (Next.js + Bootstrap)

Manutenção preditiva: indicadores, planta ao vivo, máquinas, medições, alertas, ordens de serviço,
**orçamento manutenciário** e a área do funcionário (chamados).
Next.js 14 (App Router) + React 18 + Bootstrap 5.3. JavaScript (JSX), sem TypeScript.

## Comandos
Pré-requisito: Node.js 18.17 ou superior.

```bash
cd predictmaint-next
npm install        # instala next, react e bootstrap
npm run dev        # abre em http://localhost:3000
```
Para produção: `npm run build` e depois `npm start`.

## Logins de demonstração
| Acesso | E-mail | Senha | Perfis liberados |
|---|---|---|---|
| **Máximo (admin)** | `admin@mantech.com.br` | `Mantech@2026` | Gestor, Técnico e Funcionário (escolha na tela de login) |
| **Normal (funcionário)** | `funcionario@mantech.com.br` | `Func@2026` | Só Funcionário (abre e acompanha chamados) |

Na tela de login há botões "Usar" que preenchem os campos. Também funciona a matrícula `adm001` / `fun001`.
Esses logins existem só no `lib/mock.jsx` (modo demonstração). Com a API real, senha e permissão
são validadas no backend (guarde hash, nunca senha pura) e o mock deixa de ser usado.
Para zerar os dados cadastrados na demo, apague as chaves `pm_mock_v3` e `pm_user` em DevTools > Application > Local Storage.

## Telas (rotas)
`/` login · `/dashboard` (clique nos cards para ampliar) · `/planta` · `/maquinas` · `/medicoes` · `/alertas` ·
`/ordens` (arraste cartões entre colunas) · `/orcamento` · `/chamado` e `/meus-chamados` (funcionário)

## Ligando no banco (as "portas do fetch")
Toda tela fala com `lib/api.jsx` (`api.*`). Nenhuma chama `fetch` direto.

1. Copie `.env.local.example` para `.env.local` e preencha `NEXT_PUBLIC_API_BASE`, por exemplo `http://localhost:3333/api`. Reinicie o `npm run dev`.
2. Implemente as rotas abaixo. O token do login é enviado em `Authorization: Bearer ...`.
3. Quando não precisar mais da demo, apague `lib/mock.jsx` e o import dele em `lib/api.jsx`.
4. Habilite CORS na API se ela estiver em outra origem.

Erros: responda com status 4xx/5xx e `{ "mensagem": "texto para o usuário" }`. A tela mostra a mensagem.

| Método e rota | Corpo / parâmetros | Resposta |
|---|---|---|
| `POST /auth/login` | `{perfil, usuario, senha}` | `{token, usuario:{nome, perfil, setor}}` |
| `POST /auth/recuperar` | `{usuario}` | `{ok:true}` |
| `GET /busca?q=` | | `[{tipo, titulo, sub, href}]` |
| `GET /dashboard` | | `{kpis[], vibracao{maquina,grandeza,unidade,limite,pontos[{x,y}]}, alertas[], todosAlertas[], saude[], osPorStatus{abertas,andamento,peca,concluidas,vencidas,proximas[]}, maquinas[{id,nome}]}` |
| `GET /planta/resumo` | | `{criticos, atencao, normais}` |
| `GET /planta?horas=-24..0` | | `{horas, zonas:[{nome, maquinas:[{id,nome,saude,status}]}]}` |
| `GET /planta/:id/preditivo` | | `{id,nome,status,rulDias,confianca,causas[{nome,pct}]}` |
| `GET /maquinas?setor&status&ordem` | `ordem`: saude, codigo, setor | `[{id,nome,setor,saude,status,ultima,prox,custoParada}]` |
| `GET /maquinas/:id` | | máquina + `medicoes[]` + `ordens[]` |
| `POST /maquinas` | `{id,nome,setor,prox,custoParada}` | máquina criada |
| `GET /medicoes?maquina&limite` | | `[{id,data,maquina,grandeza,nome,valor,unidade,situacao}]` |
| `GET /medicoes/serie?maquina&grandeza` | grandeza: vibracao, temperatura, corrente | `{maquina,grandeza,nome,unidade,limite,atencao,pontos[{x,y}]}` |
| `POST /medicoes` | `{maquina,grandeza,valor,unidade,data,obs}` | `{medicao, situacao, alertaCriado}` |
| `GET /alertas?setor` | | `[{id,nivel,titulo,ativo,ha,estado,medido,limite,previsao,acao,setor,maquina}]` |
| `PATCH /alertas/:id/reconhecer` | | alerta atualizado |
| `GET /limites`, `PUT /limites` | `{vibracao:{atencao,limite}, ...}` | limites |
| `GET /ordens?equipe&prioridade&periodo` | | `[{id,titulo,ativo,equipe,prazo,prioridade,status,vencida}]` (status: aberta, andamento, peca, concluida) |
| `POST /ordens` | `{titulo,ativo,equipe,prioridade,prazo,descricao,alertaId}` | OS criada |
| `PATCH /ordens/:id` | `{status}` | OS atualizada |
| `GET /orcamento/base/:maquinaId` | | `{maquina,nome,custoParadaHora,valorHora,tecnicos,horas,horasParada,pecas[{nome,qtd,valor}],terceiros,contingencia,fatores{mo,pecas,parada}}` |
| `GET /orcamento/resumo` | | `[{maquina,nome,setor,saude,status,preventivo,corretivo,risco}]` |
| `GET /orcamento`, `POST /orcamento` | `{maquina,tipo,total,cuidarAgora,esperarQuebrar,detalhes}` | orçamentos salvos |
| `GET /chamados` | | `[{id,maquina,problema,situacao,aberto,comentarios[],timeline[{nome,estado,quando,detalhe}]}]` |
| `POST /chamados` | JSON `{maquina,problema,urgencia,descricao,setor}` ou `FormData` com `dados` (JSON) + `foto` | chamado criado |
| `POST /chamados/:id/comentarios` | `{texto}` | chamado atualizado |

### Fórmula do orçamento (`lib/calc.jsx`)
`mão de obra = técnicos × horas × valor/hora` · `peças = Σ qtd × valor` · `parada = horas paradas × custo/hora parada`
· `total = (soma + terceiros) × (1 + margem%)`. A versão "esperar quebrar" aplica os `fatores`
(hora extra ×1,5, peça urgente ×1,4, parada ×3). Ajuste esses números por máquina no banco.

## Estrutura
```
app/layout.jsx, globals.css       layout raiz, Bootstrap e tema (amarelo Mantech + grafite)
app/page.jsx                        login
app/<tela>/page.jsx                uma pasta por tela
components/Painel.jsx              menu lateral, topo, busca, guarda de rota, área do funcionário
components/ui.jsx                  Modal, Toast, gráfico SVG, medidor, hook useLoad
components/AtivoModal.jsx          detalhe de máquina
lib/api.jsx                        <- TODAS as chamadas (portas do fetch)
lib/mock.jsx                       dados e logins de demonstração
lib/calc.jsx                       conta do orçamento
public/mantech-logo.png
.env.local.example                <- URL da API
```
