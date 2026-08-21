# Absolut Cinema

Site de filmes onde qualquer visitante pode ver o catálogo, usuários
cadastrados podem comentar e dar nota (1 a 5) para cada filme, e a home
mostra automaticamente **quais são os filmes mais relevantes** com base
nessas avaliações.

Este projeto parte do esqueleto original `Absolut-Cinema` (Spring Boot +
Angular, ambos praticamente vazios) e foi todo implementado usando **apenas
esses dois frameworks**. Algumas ideias de organização e de fluxo foram
inspiradas no segundo projeto que você enviou (`pi-2025`, feito em Play
Framework + Bootstrap), mas sem usar nenhuma linha de Play ou Bootstrap —
elas foram reescritas com as ferramentas do Spring/Angular. Os pontos
específicos que vieram de lá estão comentados no código-fonte e listados
mais abaixo.

---

## 1. Arquitetura

```
project/
├── Backend/    -> API REST em Spring Boot (Java)
└── frontend/   -> SPA em Angular
```

Os dois rodam como processos separados: o Angular consome a API do Spring
Boot via HTTP (`http://localhost:8080/api/...`) durante o desenvolvimento.

### 1.1 Backend (Spring Boot)

| Camada | Pacote | Responsabilidade |
|---|---|---|
| Model | `model` | Entidades JPA: `Usuario`, `Filme`, `Comentario`, enum `Perfil` |
| Repository | `repository` | Interfaces `JpaRepository` (Spring Data gera as queries) |
| Service | `service` | Regras de negócio: CRUD de filmes, cálculo de relevância, criação/remoção de comentários |
| Controller | `controller` | Endpoints REST (`/api/...`) e tratamento global de erros |
| Security | `security` | Autenticação por sessão com Spring Security |
| Config | `config` | `DataSeeder`: popula o banco com dados de exemplo ao iniciar |
| DTO | `dto` | Objetos de entrada/saída da API (nunca expomos a entidade JPA direto) |

**Banco de dados:** H2 em memória (`spring.datasource.url=jdbc:h2:mem:...`),
zero configuração necessária. Os dados são recriados a cada `mvn spring-boot:run`
pelo `DataSeeder`, que cadastra 6 filmes, 3 usuários (um deles ADMIN) e
vários comentários de exemplo. Para trocar por PostgreSQL/MySQL em produção,
basta editar `application.properties`.

**Autenticação:** baseada em sessão (cookie `JSESSIONID`), com senhas
criptografadas via BCrypt. O Angular sempre manda `withCredentials: true`
para que o cookie seja enviado/recebido. Não usamos JWT para manter o
fluxo simples e didático — é o equivalente ao que o Play Framework faz
"guardando o usuário logado na sessão HTTP".

**Cálculo de relevância** (`FilmeService.calcularRelevancia`): em vez de
ordenar filmes só pela média de notas (o que faria um filme com 1 nota 5
aparecer acima de um filme com 200 notas 4.8), usamos uma média ponderada
— a mesma lógica do "Top 250" do IMDB:

```
relevancia = (v / (v + m)) * R + (m / (v + m)) * C

R = nota média do filme
v = número de comentários/avaliações do filme
m = mínimo de avaliações para o filme "contar" com peso total (fixado em 3)
C = nota média geral de todos os filmes da base
```

Filmes sem nenhuma avaliação ficam com relevância 0 e não aparecem no topo
do ranking até receberem comentários.

**Regras de acesso** (`SecurityConfig`):
- Ver filmes, ranking e comentários → público, sem login.
- Comentar/avaliar → precisa estar logado (`ROLE_USUARIO` ou `ROLE_ADMIN`).
- Cadastrar, editar ou remover filme → só `ROLE_ADMIN`.
- Remover um comentário → o próprio autor do comentário, ou um admin.

### 1.2 Frontend (Angular, standalone components + signals)

```
frontend/src/app/
├── core/
│   ├── models/        -> interfaces TypeScript (Filme, Usuario, Comentario...)
│   ├── services/       -> AuthService, FilmeService, ComentarioService (HttpClient)
│   └── guards/          -> authGuard, adminGuard (protegem rotas)
├── components/
│   ├── navbar/          -> menu superior, mostra login/usuário
│   ├── home/             -> catálogo com busca e ordenação
│   ├── ranking/           -> lista dos filmes mais relevantes
│   ├── filme-detalhe/      -> sinopse, notas, comentários, formulário de avaliação
│   ├── login/ e registro/   -> autenticação
│   └── admin-filme-form/     -> cadastro de filme (só admin)
├── app.routes.ts    -> mapeamento de rotas
├── app.config.ts    -> providers globais (Router, HttpClient)
└── app.ts / app.html -> shell da aplicação (navbar + <router-outlet>)
```

O estado de autenticação vive em um `signal` dentro do `AuthService`
(`usuario`, `estaLogado`, `ehAdmin` como `computed`), lido de forma reativa
pela navbar e pelos guards de rota.

---

## 2. Ideias trazidas do projeto `pi-2025` (Play Framework)

O `pi-2025` é um sistema de gestão de projetos, não de filmes — mas alguns
padrões de arquitetura dele foram reaproveitados aqui, reimplementados em
Spring/Angular:

1. **Perfil de usuário como enum + verificação de admin.** O `pi-2025` usava
   um enum `Perfil` e a anotação `@Administrador` para bloquear ações.
   Recriamos isso com o enum `Perfil` (`USUARIO`/`ADMIN`) virando uma
   `ROLE_` do Spring Security no backend, e com o `adminGuard` no Angular.
2. **Login guardado em sessão.** Em vez de JWT, tanto o `pi-2025` quanto
   este projeto autenticam via sessão HTTP — aqui usando
   `HttpSessionSecurityContextRepository` do Spring Security.
3. **Seed de dados ao iniciar a aplicação.** O `pi-2025` tinha um Job
   `Inicializador.java` rodando em `@OnApplicationStart`. Recriamos o
   mesmo comportamento com `DataSeeder implements CommandLineRunner`.
4. **Separação clara de camadas** (model / repository / controller /
   segurança), já era o padrão do `pi-2025` e foi mantida aqui no estilo
   idiomático do Spring (`service` explícito, DTOs em vez de expor a
   entidade JPA direto nas respostas).

Nenhum código Play/Bootstrap foi copiado — apenas os conceitos acima.

---

## 3. Como rodar o projeto

### Pré-requisitos
- Java 21+
- Maven (ou use o `mvnw`/`mvnw.cmd` incluso, se presente)
- Node.js 20+ e npm

### 3.1 Backend

```bash
cd Backend
./mvnw spring-boot:run
```

A API sobe em `http://localhost:8080`. Na primeira execução, o
`DataSeeder` popula o banco H2 automaticamente com filmes e usuários de
exemplo (nenhuma ação manual necessária). Para inspecionar o banco, acesse
`http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:mem:absolutcinema`,
usuário `sa`, senha em branco).

**Usuários de teste já cadastrados:**
| Papel | Email | Senha |
|---|---|---|
| Administrador | `admin@absolutcinema.com` | `admin123` |
| Usuário comum | `joao@email.com` | `123456` |
| Usuário comum | `maria@email.com` | `123456` |

### 3.2 Frontend

Em outro terminal:

```bash
cd frontend
npm install
npm start
```

A aplicação abre em `http://localhost:4200` e já está configurada
(`core/api-config.ts`) para conversar com o backend em
`http://localhost:8080/api`.

### 3.3 Fluxo de uso

1. Abra `http://localhost:4200` — a home lista o catálogo de filmes.
2. Clique em um filme para ver sinopse e comentários.
3. Faça login (ou crie uma conta) para poder comentar e dar nota.
4. Acesse **"Mais relevantes"** no menu para ver o ranking calculado pela
   média ponderada.
5. Logado como `admin@absolutcinema.com`, aparece a opção **"Cadastrar
   filme"** no menu, e o botão **"Remover filme"** na página de detalhe.

---

## 4. Principais endpoints da API

| Método | Rota | Acesso |
|---|---|---|
| GET | `/api/filmes` (`?titulo=`) | público |
| GET | `/api/filmes/ranking?limite=10` | público |
| GET | `/api/filmes/{id}` | público |
| POST / PUT / DELETE | `/api/filmes` `/api/filmes/{id}` | admin |
| GET | `/api/filmes/{id}/comentarios` | público |
| POST | `/api/filmes/{id}/comentarios` | usuário logado |
| DELETE | `/api/comentarios/{id}` | autor do comentário ou admin |
| POST | `/api/auth/registrar` | público |
| POST | `/api/auth/login` | público |
| POST | `/api/auth/logout` | usuário logado |
| GET | `/api/auth/me` | usuário logado |
