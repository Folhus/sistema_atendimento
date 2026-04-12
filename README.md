# Sistema de Agendamento - Consultório Podológico

API REST em Java com Spring Boot para gerenciamento de agendamentos e pacientes de um consultório podológico. Desenvolvida para ser consumida por um bot de WhatsApp (Node.js + Baileys) e por um site administrativo.

---

## Arquitetura Geral

```
[Bot WhatsApp]          [Site Administrativo]
  Node.js + Baileys           (futuro)
       |                          |
       +----------+---------------+
                  |
            HTTP (REST)
                  |
       [API Java - Spring Boot]
          porta 8080
                  |
           [PostgreSQL]
```

O bot interpreta as mensagens do paciente, monta um payload JSON e envia para esta API. A API processa, persiste no banco e retorna uma resposta JSON que o bot repassa ao paciente.

---

## Tecnologias

- Java 17
- Spring Boot 3.2
- Spring Data JPA + Hibernate
- PostgreSQL
- Maven

---

## Pré-requisitos

- JDK 17 ou superior
- Maven 3.8 ou superior
- PostgreSQL 14 ou superior rodando localmente ou em servidor

---

## Configuração

### 1. Banco de dados

Crie o banco no PostgreSQL:

```sql
CREATE DATABASE podologia;
```

### 2. application.properties

Edite o arquivo `src/main/resources/application.properties` com suas credenciais:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/podologia
spring.datasource.username=seu_usuario
spring.datasource.password=sua_senha
```

As tabelas são criadas automaticamente pelo Hibernate na primeira execução (`ddl-auto=update`). Em produção, recomenda-se trocar para `validate` após a estrutura estar estável.

### 3. Compilar e rodar

```bash
mvn clean install
mvn spring-boot:run
```

O servidor sobe na porta `8080` por padrão.

---

## Endpoints da API

Todos os endpoints recebem e devolvem JSON. A URL base é `http://localhost:8080`.

### Agenda

| Método | Rota | Descrição |
|--------|------|-----------|
| POST | `/agenda/criar` | Cria uma nova sessão |
| POST | `/agenda/cancelar` | Cancela uma sessão existente |
| POST | `/agenda/confirmar` | Confirma presença em uma sessão |
| POST | `/agenda/concluir/{id}` | Marca a sessão como concluída |
| GET | `/agenda/status?dataHorario=` | Consulta o status de uma sessão |
| GET | `/agenda/listar?chatId=` | Lista todas as sessões de um cliente |

### Cliente

| Método | Rota | Descrição |
|--------|------|-----------|
| POST | `/cliente/registrar` | Registra ou reconhece um paciente |
| GET | `/cliente/{chatId}` | Busca dados de um paciente |
| PUT | `/cliente/{chatId}` | Atualiza dados de um paciente |

---

## Formato das Requisições

### Corpo padrão (Entrada)

A maioria dos endpoints POST espera o seguinte JSON:

```json
{
  "chatId": "5584999990000@s.whatsapp.net",
  "nome": "Maria Silva",
  "contato": "84999990000",
  "data": "2026/04/15",
  "hora": "14:30",
  "cidade": "Mossoró",
  "procedimento": "Tratamento de Unhas"
}
```

O campo `chatId` é obrigatório em todas as requisições. Os demais campos variam por endpoint.

### Formato de data

O campo `data` segue o padrão `AAAA/MM/DD` e `hora` segue `HH:MM`. Internamente são combinados no inteiro `dataHorario` no formato `AAAAMMDDHHMM` (ex: `202604151430` = 15/04/2026 às 14:30).

### Resposta padrão (Saida)

```json
{
  "sucesso": true,
  "mensagem": "Sessao criada com sucesso",
  "sessao": {
    "id": 1,
    "dataHorario": 202604151430,
    "dataFormatada": "15/04/2026 14:30",
    "procedimento": "Tratamento de Unhas",
    "cidade": "Mossoró",
    "confirmado": false,
    "completo": false
  }
}
```

Em caso de erro, `sucesso` será `false` e `mensagem` conterá a descrição do problema.

---

## Integração com o Bot (Node.js + Baileys)

O bot deve fazer requisições HTTP para esta API. Exemplo de chamada para criar agendamento:

```js
const response = await fetch('http://localhost:8080/agenda/criar', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    chatId: msg.key.remoteJid,
    nome: nomeDoContato,
    data: dataSelecionada,
    hora: horaSelecionada,
    cidade: cidadeSelecionada,
    procedimento: procedimentoSelecionado
  })
});

const dados = await response.json();
await sock.sendMessage(msg.key.remoteJid, { text: dados.mensagem });
```

Fluxo recomendado ao iniciar qualquer conversa:

1. Chamar `POST /cliente/registrar` com o `chatId` do usuário
2. A API reconhece o paciente existente ou o cadastra automaticamente
3. Prosseguir com as ações de agendamento

---

## Estrutura do Projeto

```
src/
  main/
    java/sistema/
      Sistema.java              - Ponto de entrada (Spring Boot)
      Agenda.java               - Logica de agendamento
      ClienteService.java       - Logica de gerenciamento de pacientes
      AgendaController.java     - Endpoints REST de agenda
      ClienteController.java    - Endpoints REST de clientes
      GlobalExceptionHandler.java - Tratamento centralizado de erros
      Impressora.java           - Geracao de logs em arquivo
      Cliente.java              - Entidade JPA: paciente
      Sessao.java               - Entidade JPA: consulta agendada
      Procedimento.java         - Entidade JPA: tipo de procedimento
      Atendimento.java          - Entidade JPA: registro pos-consulta
      Repositorios.java         - Interfaces de acesso ao banco (JPA)
      Entrada.java              - DTO de entrada (dados recebidos)
      Saida.java                - DTO de saida (dados retornados)
    resources/
      application.properties   - Configuracoes da aplicacao
  test/
    java/sistema/
      AppTest.java              - Testes basicos
    resources/
      application-test.properties - Configuracao H2 para testes
```

---

## Testes

Os testes utilizam banco H2 em memória e não exigem PostgreSQL instalado:

```bash
mvn test
```

---

## Regras de Negócio

- Não é possível agendar uma sessão dentro de um intervalo de 20 minutos de uma sessão já existente
- O `chatId` do WhatsApp é o identificador único de cada paciente
- Ao cancelar uma sessão, ela é removida do banco junto com o vínculo ao cliente
- Uma sessão pode ser confirmada pelo próprio paciente via bot
- A conclusão da sessão é feita pelo painel administrativo (uso interno)

---

## Geração de Logs

Para gerar um relatório completo do banco em arquivo `.txt`:

Chame internamente `Impressora.imprimirBanco()`. Os arquivos são salvos na pasta `logs/` na raiz do projeto com timestamp no nome.

---

## Variáveis de Ambiente (recomendado para produção)

Em vez de deixar credenciais no `application.properties`, use variáveis de ambiente:

```properties
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASS}
```

E exporte antes de rodar:

```bash
export DB_URL=jdbc:postgresql://localhost:5432/podologia
export DB_USER=postgres
export DB_PASS=sua_senha
mvn spring-boot:run
```

---

## Licença

Uso privado. Todos os direitos reservados.
