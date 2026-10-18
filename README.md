# Sistema Escolar — Spring Boot

**Nome:** RUIMAR CLETO DE ALBUQUERQUE SILVANO
**Disciplina:** Programação Web 1 · Senac Rio — Ciclo Avaliativo 1 (*Caça aos Erros*)

## Descrição

API REST de um sistema escolar de cadastro, feita com **Java 17**, **Spring Boot 4.0.5**, **Spring Data JPA**, **Maven** e banco **H2 em memória**. Ela cadastra **professores** e **alunos** e registra as **matrículas** dos alunos nas turmas dos professores (turno, semestre e observação).

Este repositório contém o projeto base **com os 8 erros corrigidos**. Cada erro está documentado abaixo: arquivo, o que estava errado, como foi descoberto e qual foi a correção.

---

## Detalhamento dos erros

### Erro 1 — H2 fora do classpath de execução

**Arquivo:** `pom.xml`

**O que estava errado:** a dependência `com.h2database:h2` estava com `<scope>test</scope>`. Esse escopo só coloca o driver no classpath dos *testes*; ao rodar a aplicação (`spring-boot:run`) o H2 não existe.

**Como descobri:** lendo o `pom.xml` (era o único `scope` fora do padrão) junto com o `application-test.properties`, que aponta para `jdbc:h2:mem:...`.

**Sintoma:** a aplicação não sobe. O Spring falha ao criar o `DataSource` por não achar a classe do driver (mensagem do tipo `Failed to load driver class org.h2.Driver ...`).

**Correção:**

```xml
<!-- antes -->
<scope>test</scope>

<!-- depois -->
<scope>runtime</scope>
```

### Erro 2 — Tabelas nunca criadas

**Arquivo:** `src/main/resources/application-test.properties`

**O que estava errado:** `spring.jpa.hibernate.ddl-auto=none` manda o Hibernate *não* gerar o schema. Como o H2 em memória sobe vazio e o projeto não tem Flyway nem `schema.sql`, as tabelas `professores`, `alunos` e `matriculas` nunca existem. O perfil ativo é o `test` (`application.properties`) e é nele que essa configuração vale; o `application-dev.properties` está vazio e não é usado.

**Como descobri:** lendo o `application-test.properties` junto com o texto da atividade ("o H2 em memória é recriado vazio toda vez que a aplicação sobe") e confirmando que não há outra forma de criar as tabelas no projeto.

**Sintoma:** a aplicação sobe, mas qualquer chamada que acessa o banco responde 500 com algo como `Table "ALUNOS" not found (this database is empty)`.

**Correção:**

```properties
# antes
spring.jpa.hibernate.ddl-auto=none

# depois
spring.jpa.hibernate.ddl-auto=create-drop
```

Com `create-drop`, o Hibernate cria as tabelas a partir das entidades ao subir e as remove ao encerrar.

### Erro 3 — Rota errada no ProfessorController

**Arquivo:** `src/main/java/com/github/app/controller/ProfessorController.java`

**O que estava errado:** `@RequestMapping("/professor")`, no singular. Os endpoints da atividade são `/professores`, no plural, como `/alunos` e `/matriculas`.

**Como descobri:** comparando as anotações dos três controllers com a lista de endpoints da atividade.

**Sintoma:** nenhuma rota `/professores` existia: `GET /professores` responde 404 (`No static resource professores`) e `POST`, `PUT` e `DELETE` também falham (em geral com 405, método não suportado). Só `/professor` funcionava.

**Correção:**

```java
// antes
@RequestMapping("/professor")

// depois
@RequestMapping("/professores")
```

### Erro 4 — Professor sem geração automática de id

**Arquivo:** `src/main/java/com/github/app/model/professor/Professor.java`

**O que estava errado:** o campo `id` tinha só `@Id`, sem `@GeneratedValue`. O id de professor deve ser gerado pelo banco, como já acontece em `Aluno` e `Matricula`.

**Como descobri:** comparando a entidade `Professor` com `Aluno` e `Matricula`, que usam `@GeneratedValue(strategy = GenerationType.IDENTITY)`.

**Sintoma:** `POST /professores` responde 500 com `IdentifierGenerationException`: o Hibernate exige que o id seja atribuído manualmente antes de salvar.

**Correção:**

```java
// antes
@Id
private Integer id;

// depois
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Integer id;
```

### Erro 5 — RA recebendo o curso

**Arquivo:** `src/main/java/com/github/app/model/aluno/Aluno.java` (construtor `Aluno(DadosCadastroAluno)`)

**O que estava errado:** `this.ra = dados.curso();`. O campo `ra` era preenchido com o valor do curso.

**Como descobri:** lendo o construtor linha a linha e comparando com o DTO `DadosCadastroAluno`, que tem `ra` e `curso` como campos separados.

**Sintoma:** cadastrando `"ra": "2026001"` e `"curso": "ADS"`, o `GET /alunos/listar` devolve `"ra": "ADS"`. A resposta é 200, mas o dado está no campo errado.

**Correção:**

```java
// antes
this.ra = dados.curso();

// depois
this.ra = dados.ra();
```

### Erro 6 — PUT de aluno não grava no banco

**Arquivo:** `src/main/java/com/github/app/controller/AlunoController.java` (método `atualizar`)

**O que estava errado:** faltava `@Transactional` no `PUT`. O método altera a entidade carregada (`aluno.atualizarInformacoes(dados)`) e depende do commit da transação para o JPA gerar o `UPDATE`. Sem transação, a alteração nunca chega ao banco.

**Como descobri:** comparando com `ProfessorController.atualizar`, que tem `@Transactional`.

**Sintoma:** `PUT /alunos` com `{ "id": 1, "curso": "Engenharia de Software" }` responde 200, mas o `GET /alunos/listar` seguinte continua mostrando o curso antigo.

**Correção:**

```java
// antes
@PutMapping
public void atualizar(@RequestBody DadosAtualizacaoAluno dados) { ... }

// depois
@PutMapping
@Transactional
public void atualizar(@RequestBody DadosAtualizacaoAluno dados) { ... }
```

### Erro 7 — Aluno limitado a uma única matrícula

**Arquivo:** `src/main/java/com/github/app/model/matricula/Matricula.java`

**O que estava errado:** o relacionamento com `Aluno` estava como `@OneToOne`. Com `@OneToOne`, o Hibernate cria uma restrição `UNIQUE` em `aluno_id`, então cada aluno só consegue ter uma matrícula. A regra da atividade diz que o mesmo aluno pode ter várias (por exemplo, em turnos diferentes). O relacionamento correto é muitos-para-um, o mesmo já usado em `professor`.

**Como descobri:** relendo as regras da atividade e comparando as anotações de `aluno` e `professor` dentro de `Matricula`.

**Sintoma:** a 1ª matrícula do aluno funciona. A 2ª (mesmo aluno, outro turno) responde 500 com violação de índice único (algo como `Unique index or primary key violation ... MATRICULAS(ALUNO_ID ...)`).

**Correção:**

```java
// antes
@JoinColumn(name = "aluno_id")
@OneToOne
private Aluno aluno;

// depois
@JoinColumn(name = "aluno_id")
@ManyToOne
private Aluno aluno;
```

### Erro 8 — Campo do JSON com nome diferente do DTO

**Arquivos:** `src/main/java/com/github/app/model/matricula/DadosCadastroMatricula.java` e `src/main/java/com/github/app/controller/MatriculaController.java`

**O que estava errado:** o DTO declarava `Integer idAluno`, mas o JSON da atividade usa `alunoId` (no mesmo padrão de `professorId`). O Jackson ignora o campo desconhecido `alunoId`, e `idAluno` chega `null`.

**Como descobri:** comparando o JSON de exemplo de `POST /matriculas` com os componentes do record `DadosCadastroMatricula`.

**Sintoma:** `POST /matriculas` com o JSON da atividade responde 500, com algo como `The given id must not be null` (`InvalidDataAccessApiUsageException`, vindo de `alunoRepository.getReferenceById(...)`).

**Correção:** o componente do record passou a se chamar `alunoId`, e o controller usa `dados.alunoId()`.

```java
// DadosCadastroMatricula.java
// antes:  Integer idAluno,
// depois: Integer alunoId,

// MatriculaController.java
// antes:  alunoRepository.getReferenceById(dados.idAluno());
// depois: alunoRepository.getReferenceById(dados.alunoId());
```

---

## Como executar

Requisitos: **JDK 17 ou superior** e acesso à internet na primeira execução (o Maven baixa as dependências).

```bash
git clone https://github.com/<seu-usuario>/sistema_escolar_java.git
cd sistema_escolar_java
./mvnw spring-boot:run        # no Windows: mvnw.cmd spring-boot:run
```

A API sobe em `http://localhost:8080`, sem autenticação. O perfil ativo é o `test` (definido em `application.properties`): banco **H2 em memória**, recriado **vazio** a cada execução (`ddl-auto=create-drop`).

## Roteiro para testar (resultado esperado)

Use o Insomnia ou o Postman. Os ids começam em 1 porque o banco sobe vazio. Para `POST /professores` e `POST /alunos`, use os JSON de exemplo da atividade.

| Passo | Requisição | Resultado esperado |
|-------|------------|--------------------|
| 1 | `POST /professores` | 200 |
| 2 | `GET /professores/listar` | `id: 1`, `registro: "PRF-1001"`, `disciplina: "MATEMATICA"` |
| 3 | `POST /alunos` | 200 |
| 4 | `GET /alunos/listar` | `ra: "2026001"` e `curso: "ADS"`, cada dado no seu campo |
| 5 | `PUT /professores` com `{ "id": 1, "email": "ana.nova@escola.com" }`, depois `GET /professores/listar` | e-mail novo salvo |
| 6 | `PUT /alunos` com `{ "id": 1, "curso": "Engenharia de Software" }`, depois `GET /alunos/listar` | curso novo salvo |
| 7 | `POST /matriculas` com `{ "alunoId": 1, "professorId": 1, "turno": "MANHA", "semestre": "2026.2", "observacao": "Turma A" }` | 200 |
| 8 | `POST /matriculas` com o mesmo aluno e `"turno": "TARDE"` | 200 (2ª matrícula do mesmo aluno) |
| 9 | `GET /matriculas` | 2 matrículas, com `dataMatricula` preenchida automaticamente |
| 10 | `GET /professores` e `GET /alunos` (paginados), `GET /professores/todos` e `GET /alunos/todos` | 200 |
| 11 | `DELETE /matriculas/1` e `/matriculas/2`, depois `DELETE /alunos/1` e `DELETE /professores/1` | 200; listagens vazias |

Observação: no passo 11, as matrículas precisam ser removidas antes do aluno e do professor, porque elas referenciam os dois.
