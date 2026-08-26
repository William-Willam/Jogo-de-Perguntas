# Quiz de Conhecimentos Gerais — Documento de Requisitos

## 1. Visão Geral

Aplicação de quiz de conhecimentos gerais, disponível como **Web App / PWA** (Progressive Web App), permitindo instalação no celular como um aplicativo sem a necessidade de desenvolvimento mobile nativo.

## 2. Arquitetura

- **Backend**: Java + Spring Boot (API REST)
- **Frontend**: Angular, estruturado como PWA (instalável, responsivo)
- **Fonte de perguntas**: [Open Trivia Database (OpenTDB)](https://opentdb.com/), consumida pelo backend
- **Banco de dados**: MySQL

```
[ Angular PWA ] <--HTTPS/REST--> [ Spring Boot API ] <---> [ MySQL ]
                                          |
                                          v
                                   [ OpenTDB API ]
```

## 3. Requisitos Funcionais (MVP — Single Player)

| ID | Descrição |
|----|-----------|
| RF01 | Cadastro/login via Google (OAuth2) |
| RF02 | Buscar 10 perguntas aleatórias na OpenTDB (categoria e dificuldade aleatórias), buscadas todas antes do início da partida |
| RF03 | Iniciar partida individual com as 10 perguntas |
| RF04 | Exibir cronômetro de 20 segundos por pergunta |
| RF05 | Calcular pontuação por acerto, com bônus por velocidade de resposta |
| RF06 | Salvar histórico de partidas por usuário |
| RF07 | Gerar ranking global (ordenação das pontuações de todos os usuários) |
| RF08 | Exibir resultado da partida ao final (acertos, erros, pontuação total) |

## 4. Requisitos Não Funcionais

| ID | Descrição |
|----|-----------|
| RNF01 | API RESTful, stateless |
| RNF02 | Frontend Angular estruturado como PWA (manifest.json + service worker) |
| RNF03 | Interface responsiva (mobile-first) |
| RNF04 | As 10 perguntas da partida são buscadas da OpenTDB **antes** do início (não pergunta por pergunta), evitando falha em meio à partida |

## 5. Autenticação

A autenticação é feita **exclusivamente via login social com Google (OAuth2)**. Não há cadastro tradicional por email/senha, portanto não existe fluxo de "esqueci minha senha" — a recuperação de acesso é o próprio login com Google.

**Fluxo:**
1. Angular usa o Google Identity Services (OAuth2) para autenticar o usuário
2. Google retorna um ID Token (JWT assinado pelo Google) para o Angular
3. Angular envia o ID Token para o backend Spring Boot
4. Backend valida a assinatura do ID Token diretamente com o Google (nunca confia no frontend)
5. Se é a primeira vez desse email, backend cria o usuário no banco; senão, apenas autentica
6. Backend emite seu próprio JWT de sessão (access + refresh token), usado daí em diante em toda a aplicação

A entidade `Usuário` não possui campo de senha — possui `googleId` (o `sub` do token do Google) e email.

## 6. Requisitos de Segurança

Levantados a partir da análise de vulnerabilidades identificadas em projetos anteriores. Esta seção é tratada como requisito de primeira classe, não como item opcional.

| ID | Requisito | Justificativa |
|----|-----------|----------------|
| RS01 | Validar o ID Token do Google **sempre no backend**, nunca confiar no que o frontend informa | O Angular pode ser manipulado; o backend é a fonte da verdade |
| RS02 | Verificar se o campo `aud` (audience) do token bate com o **Client ID** da aplicação | Evita aceitar token emitido para outra aplicação |
| RS03 | Verificar `email_verified: true` no payload do token do Google | Garante que só contas com email confirmado pelo Google sejam aceitas |
| RS04 | Nunca expor o **Client Secret** no Angular (frontend) | Client Secret é só para comunicação server-to-server |
| RS05 | **Access Token (JWT próprio) de curta duração** (15–30 min) | Reduz a janela de exposição caso o token vaze |
| RS06 | **Refresh Token** de duração maior, revogável (persistido em tabela no banco) | Permite invalidar sessões remotamente sem esperar expiração do access token |
| RS07 | Token armazenado em **cookie `httpOnly` + `Secure` + `SameSite=Strict`** — nunca em `localStorage` | Protege contra roubo de token via ataques XSS |
| RS08 | Checagem de **autorização por posse do recurso** em toda rota que acessa dados de usuário (histórico, ranking pessoal) | Previne IDOR (Insecure Direct Object Reference) — usuário A acessando dados do usuário B |
| RS09 | **Validação de entrada no backend** (Bean Validation / `@Valid`) em todos os endpoints, independente da validação do Angular | Validação no front é UX; validação no backend é segurança |
| RS10 | **CORS restrito** à origem do frontend Angular, nunca `*` | Evita que sites externos façam requisições autenticadas contra a API |
| RS11 | **HTTPS obrigatório em produção** | Protege o JWT e credenciais contra interceptação em trânsito |

## 8. Estratégia de Testes

Testes fazem parte do escopo do MVP, priorizados nas camadas de maior risco: regra de negócio e segurança.

**Testes unitários — camada Service (prioridade máxima)**
- Ferramentas: JUnit 5 + Mockito (Repository mockado, lógica da Service testada isolada)
- Cobre: cálculo de pontuação com bônus de velocidade, resposta correta/incorreta, tempo esgotado contando como erro

**Testes de integração — camada Controller**
- Ferramentas: `@SpringBootTest` + `MockMvc`, com banco H2 em memória (sem depender do MySQL para rodar os testes)
- Cobre: fluxo completo requisição → Controller → Service → banco; validação de autenticação (401 sem token) e autorização (403 ao tentar acessar recurso de outro usuário — validação direta da regra anti-IDOR, RS08)

**Testes de segurança dedicados**
- Token expirado é rejeitado
- Token com `aud` diferente do Client ID é rejeitado
- Usuário tentando acessar histórico/partida de outro usuário recebe 403

**Fora do escopo do MVP**
- Testes end-to-end automatizados no frontend (Cypress/Playwright) — considerado para uma versão futura
- Cobertura de 100% — foco em regra de negócio e segurança, não em código trivial (getters/setters)

## 9. Escopo — Fora desta versão (MVP)

- Salas com convite / PvP em tempo real / WebSocket (planejado para v2)
- Ranking por sala (planejado para v2)
- Aplicativo mobile nativo (Android/iOS) — coberto via PWA
- Cliente desktop (removido na reformulação do projeto)
- Escolha manual de categoria/dificuldade pelo jogador (mantido aleatório nesta versão)
- Cadastro/login tradicional por email e senha (autenticação exclusiva via Google)

## 10. Próximos Passos

- [ ] Modelagem das entidades do banco de dados (Usuário, Partida, Resposta, Ranking)
- [ ] Estruturação inicial do projeto Spring Boot
- [ ] Configuração do Angular como PWA
- [ ] Implementação da autenticação (Google OAuth2 + JWT próprio)
- [ ] Testes unitários (Service) e de integração (Controller) desde o início do desenvolvimento
