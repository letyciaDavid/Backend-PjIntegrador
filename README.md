# jwt-api – Sistema de Gestão de Biblioteca (Back-end)

API RESTful desenvolvida em **Node.js + Express 5 + Sequelize + SQLite + JWT** para controle completo de uma biblioteca (categorias, autores, livros, leitores, atendentes, empréstimos e devoluções), utilizando autenticação stateless com **Access Token + Refresh Token (rotação e detecção de reuso)**.

---

## 1. Como instalar e rodar

```bash
cd backend-jwt-api
npm install
copy .envex .env
npm start
```

O servidor iniciará em `http://localhost:3000`.

---

## 2. Variáveis de Ambiente (`.env`)

Baseadas no arquivo `.envex`:

```env
JWT_SECRET=troque-por-uma-chave-grande-e-aleatoria
JWT_REFRESH_SECRET=outra-chave-diferente-da-primeira
JWT_EXPIRES_IN=15m
PORT=3000
CORS_ORIGIN=http://localhost:3001
```

> **Dica para o vídeo de apresentação:** Para demonstrar o token expirando e sendo renovado sozinho pelo interceptor do front-end, basta alterar `JWT_EXPIRES_IN=30s` no `.env` e reiniciar a API.

---

## 3. Usuários criados automaticamente pelo Seed (`models/index.js`)

Ao iniciar a API pela primeira vez (`db.sync()`), o seed cria automaticamente dois usuários de teste e dados iniciais do acervo:

| Perfil | E-mail | Senha | Permissões |
|---|---|---|---|
| **Administrador** | `admin@biblioteca.com` | `123456` | Acesso total (atendentes, categorias, autores, livros, leitores, empréstimos e relatórios) |
| **Atendente** | `atendente@biblioteca.com` | `123456` | Consulta ao acervo, cadastro de leitores, registro de empréstimos e devoluções |

---

## 4. Decisões Técnicas tomadas

1. **Fluxo de 2 Tokens (Access + Refresh com Rotação):**
   - O `accessToken` tem curta duração (`15m`) e carrega `{ id, name, email, isAdmin }`.
   - O `refreshToken` tem validade de `7 dias` e é persistido na tabela `RefreshTokens` com os campos `token`, `userId`, `expiresAt` e `revoked`.
   - A cada chamada em `POST /v1/auth/refresh`, o refresh token antigo é marcado como `revoked = true` e um novo par é emitido (**rotação de refresh token**).
   - **Detecção de roubo de token:** Se um refresh token já revogado for enviado novamente no `/v1/auth/refresh`, a API revoga imediatamente **todos** os refresh tokens daquele usuário.
2. **Manutenção da tabela `logout_list` junto com `RefreshToken`:**
   - Optamos por **manter** a tabela `logout_list` para os `accessTokens` revogados no momento do logout, além de revogar o `refreshToken` na tabela `RefreshTokens`.
   - **Justificativa:** Como o JWT de acesso ainda pode ter até 15 minutos restantes de vida útil no momento em que o usuário clica em "Sair", manter o `accessToken` na `logout_list` garante revogação instantânea da sessão ativa, enquanto a tabela `RefreshTokens` impede qualquer renovação futura.
3. **Transações do Sequelize (`db.transaction`)**:
   - As operações de empréstimo (`POST /v1/loans`) e devolução (`PATCH /v1/loans/:id/return`) rodam dentro de transações do Sequelize para garantir que o estoque (`availableCopies`) nunca fique inconsistente caso alguma regra de negócio falhe.
4. **Coleção para Testes:**
   - O arquivo `postman-collection.json` na raiz do projeto contém todas as requisições prontas para importação no Postman, Insomnia ou Thunder Client.

