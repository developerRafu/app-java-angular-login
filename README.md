# Documentação de Especificação - Aplicação Login/Registro

## 1. VISÃO GERAL DO PROJETO

### 1.1 Objetivo
Desenvolver uma aplicação web completa com autenticação de usuários utilizando Spring Boot no backend e Angular no frontend, com banco de dados PostgreSQL rodando em container Docker.

### 1.2 Tecnologias Obrigatórias
- **Backend**: Java 17+, Spring Boot, Spring Security, JWT, JPA/Hibernate, PostgreSQL
- **Frontend**: Angular 15+, TypeScript, Bootstrap/Tailwind para estilização
- **Infraestrutura**: Docker, Docker Compose
- **Testes**: JUnit 5, Mockito (backend)

---

## 2. ESTRUTURA DO PROJETO

### 2.1 Diretórios e Organização
```
project-root/
├── backend/
│   ├── src/main/java/com/app/
│   │   ├── controller/     # Controllers REST
│   │   ├── service/        # Regras de negócio
│   │   ├── repository/     # Interface com banco de dados
│   │   ├── model/          # Entidades JPA
│   │   ├── dto/            # Data Transfer Objects
│   │   ├── config/         # Configurações (Security, JWT, CORS)
│   │   ├── exception/      # Exceções customizadas
│   │   └── util/           # Classes utilitárias
│   ├── src/test/java/      # Testes unitários
│   ├── pom.xml             # Dependências Maven
│   └── Dockerfile          # Configuração do container backend
├── frontend/
│   ├── src/app/
│   │   ├── components/     # Componentes Angular
│   │   │   ├── login/      # Página de login
│   │   │   └── register/   # Página de registro
│   │   ├── services/       # Serviços (Auth, API)
│   │   ├── models/         # Interfaces/Modelos
│   │   ├── guards/         # Guards de rota
│   │   ├── interceptors/   # Interceptores HTTP
│   │   └── app-routing.module.ts
│   ├── angular.json
│   └── Dockerfile
├── docker-compose.yaml
└── README.md
```

---

## 3. REQUISITOS DO BACKEND

### 3.1 Modelo de Dados (User)

**Tabela: users**
- `id`: BIGINT, PRIMARY KEY, AUTO_INCREMENT
- `name`: VARCHAR(255), NOT NULL
- `email`: VARCHAR(255), UNIQUE, NOT NULL
- `password`: VARCHAR(255), NOT NULL (armazenar hash BCrypt)
- `created_at`: TIMESTAMP
- `updated_at`: TIMESTAMP

### 3.2 Endpoints API

| Método | Endpoint | Descrição | Autenticação |
|--------|----------|-----------|--------------|
| POST | /api/auth/login | Autenticar usuário | Não |
| POST | /api/users/register | Criar novo usuário | Não |
| GET | /api/users/me | Obter dados do usuário logado | Sim |

**Observações:**
- NUNCA retornar a senha nos responses
- Todas as rotas com exceção de login/register devem exigir token JWT

### 3.3 Regras de Negócio

**Registro de Usuário:**
- Validar se email já existe no sistema
- Senha deve ter no mínimo 8 caracteres
- Confirmar senha deve ser igual à senha
- Gerar hash da senha com BCrypt (salt rounds 10)
- Preencher campos created_at e updated_at automaticamente

**Login:**
- Buscar usuário por email
- Verificar senha com BCrypt
- Gerar token JWT com expiração de 24 horas
- Retornar token + dados básicos do usuário

### 3.4 Configurações do Spring Boot

**application.properties:**
```
spring.application.name=backend-auth
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5432/userdb}
spring.datasource.username=${DB_USERNAME:admin}
spring.datasource.password=${DB_PASSWORD:admin123}
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

jwt.secret=${JWT_SECRET:chaveSuperSecretaParaJWT123}
jwt.expiration=86400000

spring.jackson.time-zone=America/Sao_Paulo
```

### 3.5 Tratamento de Erros

- Usar `@ControllerAdvice` para tratamento global
- Padronizar resposta de erro com:
  - `timestamp`
  - `status`
  - `message`
  - `errors` (para validações de campo)

---

## 4. REQUISITOS DO FRONTEND

### 4.1 Páginas

**Login (/login):**
- Formulário com campos: email, senha
- Botão "Entrar"
- Link para página de registro
- Exibir mensagens de erro (usuário não encontrado, senha incorreta)

**Registro (/register):**
- Formulário com campos: nome, email, senha, confirmar senha
- Validações em tempo real
- Botão "Cadastrar"
- Link para página de login
- Mensagem de sucesso após cadastro

**Dashboard (/dashboard):**
- Página protegida por autenticação
- Exibir nome do usuário logado
- Botão "Sair" (logout)

### 4.2 Regras de Validação (Frontend)

**Campos Obrigatórios:**
- Todos os campos de ambos os formulários são obrigatórios

**Email:**
- Deve estar no formato válido (validação com regex)

**Senha:**
- Mínimo 8 caracteres
- Pode ser configurado para exigir maiúscula/minúscula/número

**Confirmar Senha:**
- Deve ser igual ao campo senha

### 4.3 Interceptor HTTP

- Adicionar token JWT no header Authorization: `Bearer {token}`
- Se token expirado, redirecionar para login
- Tratar erros de autenticação (401)

### 4.4 Guard de Rotas

- Proteger rotas que exigem autenticação
- Verificar se token existe no localStorage
- Redirecionar para /login se não autenticado

---

## 5. DOCKER CONFIGURAÇÃO

### 5.1 docker-compose.yaml

**Deve conter 3 serviços:**

1. **PostgreSQL (postgres:15-alpine)**
   - Porta: 5432
   - Banco: userdb
   - Usuário: admin
   - Senha: admin123
   - Volume persistente para dados
   - Healthcheck configurado

2. **Backend (Spring Boot)**
   - Build: ./backend
   - Porta: 8080
   - Variáveis de ambiente para conexão com banco
   - Depender do PostgreSQL estar saudável

3. **Frontend (Angular)**
   - Build: ./frontend
   - Porta: 4200 (mapeada para 80)
   - Depender do backend

### 5.2 Dockerfiles

**Backend Dockerfile:**
- Multi-stage build
- Utilizar maven:3.8-openjdk-17 para build
- Utilizar openjdk:17-jre-slim para runtime
- Copiar JAR da pasta target

**Frontend Dockerfile:**
- Utilizar node:18-alpine para build
- Utilizar nginx:alpine para servir
- Copiar arquivos dist/

---

## 6. TESTES UNITÁRIOS

### 6.1 Backend (Spring Boot)

**Obrigatório testar:**
- `UserService.registerUser()` - cenários de sucesso e falha
- `UserService.authenticate()` - cenários de sucesso e falha
- `UserRepository` - métodos customizados
- Controllers com MockMvc
- Validações de DTOs

**Cobertura mínima:** 80%

### 6.2 Frontend (Angular)

**Obrigatório testar:**
- `AuthService` - registro e login
- `LoginComponent` - submissão do formulário
- `RegisterComponent` - validações do formulário
- Guard de rotas
- Interceptor HTTP

**Cobertura mínima:** 80%

---

## 7. SEGURANÇA

### 7.1 Backend Security
- Configurar Spring Security com CORS habilitado (origem: http://localhost:4200)
- CSRF desabilitado (stateless)
- Filtro JWT configurado
- BCrypt para hash de senhas
- Mapeamento de permissões por role (admin/user)

### 7.2 Frontend Security
- Armazenar token no localStorage (ou sessionStorage)
- Limpar token no logout
- Interceptor para adicionar token
- Logout automático em caso de token inválido

---

## 8. VARIÁVEIS DE AMBIENTE

### Backend
```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
JWT_SECRET
```

### Frontend
```
API_URL (URL do backend)
```

---

## 9. PASSOS DE IMPLEMENTAÇÃO

### Backend (Ordem sugerida):
1. Criar projeto Spring Boot com dependências
2. Configurar application.properties
3. Criar entidade User
4. Criar repository (JPA)
5. Criar DTOs
6. Criar service com regras de negócio
7. Criar controller
8. Configurar Security (JWT, BCrypt, CORS)
9. Implementar tratamento de exceções
10. Criar testes unitários

### Frontend (Ordem sugerida):
1. Criar projeto Angular
2. Criar models (interfaces)
3. Criar AuthService
4. Implementar interceptors
5. Criar LoginComponent
6. Criar RegisterComponent
7. Criar DashboardComponent
8. Configurar rotas e guards
9. Estilizar com framework CSS
10. Criar testes unitários

### Infraestrutura:
1. Criar Dockerfile backend
2. Criar Dockerfile frontend
3. Configurar docker-compose.yaml
4. Testar execução completa

---

## 10. CRITÉRIOS DE ACEITE

✅ Backend inicia sem erros no Docker  
✅ Frontend inicia sem erros no Docker  
✅ É possível criar um novo usuário  
✅ É possível fazer login com o usuário criado  
✅ Senhas são armazenadas com hash  
✅ Token JWT é gerado e validado  
✅ Rotas protegidas funcionam  
✅ Mensagens de erro são claras  
✅ Testes unitários passam  
✅ Código segue padrões (não usar IA)  

---

## 11. ENTREGÁVEIS

- Código fonte completo do backend
- Código fonte completo do frontend
- docker-compose.yaml
- Dockerfiles (backend e frontend)
- README com instruções de execução
- Relatório de cobertura de testes

---

## 12. RESTRIÇÕES

- **PROIBIDO** usar geradores automáticos de código
- **PROIBIDO** copiar código de outras fontes
- **PROIBIDO** usar frameworks de baixo código (ex: JHipster)
- **NECESSÁRIO** fazer commit de TODO o código
- **NECESSÁRIO** documentar decisões importantes
- **NECESSÁRIO** seguir boas práticas de OOP e Clean Code

---

**Data de Entrega:** N/A
**Responsável:** Anderson
