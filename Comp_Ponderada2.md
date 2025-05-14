# Projeto Individual (COMP Parte 2): Conectando Banco de Dados e Servidor

Objetivo: Implementar Models e Controllers, integrar um banco de dados e desenvolver endpoints para navegação dinâmica.

Entrega:
- Submissão individual.
- Projeto hospedado em repositório público no GitHub.
- Documentação atualizada (README e WAD.md).

Prazo de Entrega: até sexta-feira da *semana 5* do módulo.

---

## Recapitulando os fundamentos

### Integração com Banco de Dados

A integração com o banco de dados é feita utilizando o pacote `pg`, que permite ao Node.js executar comandos SQL diretamente no PostgreSQL.

Essa integração possibilita que o servidor:
- Leia dados salvos no banco (por exemplo, todas as tarefas);
- Insira novos registros;
- Atualize registros existentes;
- Remova dados.

A conexão é configurada com segurança usando variáveis de ambiente definidas no arquivo `.env`.

---

### Migrações

**Migração** é o processo de criação ou alteração da estrutura do banco de dados (como criar tabelas).

Você pode fazer a migração manualmente com um script JavaScript (`migrations/migrate.js`), que executa comandos SQL ao rodar:

```bash
npm run migrate
```

Ou utilziar a estrutura do mvc-boilerplate, alternato o arquivo init.sql e rodando o comando `npm run init-db`.

---

### Models

Models são responsáveis por representar a estrutura dos dados da aplicação. Neste projeto, como não usamos um ORM (como Sequelize), os Models estão embutidos diretamente nos **Controllers** por meio de comandos SQL.

---

### Controllers

Controllers são funções JavaScript que lidam com a lógica da aplicação, como:

- Receber uma requisição HTTP;
- Validar os dados recebidos;
- Executar um comando no banco (via SQL);
- Enviar uma resposta adequada para o cliente.

Exemplo de controller para criar uma nova tarefa:

```javascript
exports.criarTarefa = async (req, res) => {
  const { nome, descricao } = req.body;
  const query = 'INSERT INTO tarefas (nome, descricao) VALUES ($1, $2) RETURNING *';

  try {
    const result = await pool.query(query, [nome, descricao]);
    res.status(201).json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};
```

Esse padrão ajuda a manter o código organizado e separado por responsabilidade.

---

### Rotas

As **rotas** conectam URLs às funções dos controllers. Por exemplo, ao fazer um `GET` para `/api/tarefas`, o servidor chama o controller que lista as tarefas do banco de dados.

Exemplo de rota:

```javascript
router.get('/tarefas', TarefaController.listarTarefas);
```

As rotas estão definidas no arquivo `routes/index.js`.

---

Esses conceitos juntos formam a base de muitos sistemas web modernos, conectando o frontend (interface do usuário), backend (lógica de negócio) e banco de dados (armazenamento de informações).

## Instruções

### Etapa 1 - Integrando seu BD ao projeto
Caso ainda não tenha feito, adicione o script SQL ao arquivo init.sql e exeute o comando de criação do BD `npm run init-db`.

#### Configure o Banco de Dados 

Crie um arquivo `.env` para armazenar as variáveis de conexão com o banco de dados:

```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=seu_usuario
DB_PASSWORD=sua_senha
DB_NAME=nome_do_banco
```

Em seguida, crie um arquivo de configuração `config/database.js` para carregar essas variáveis e estabelecer a conexão:

```javascript
// config/db.js
require('dotenv').config();

const { Pool } = require('pg');

// Criando a pool de conexões
const pool = new Pool({
  user: process.env.DB_USER,
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  password: process.env.DB_PASSWORD,
  port: process.env.DB_PORT,
});

module.exports = pool;
```


### Etapa 2- Implementando Controllers e Rotas para alteração do BD

#### Passo 1

Crie o arquivo `controllers/TarefaController.js` para manipular os dados da tabela **tarefas**.

```javascript
// controllers/TarefaController.js
const pool = require('../config/database');

// Criar uma nova tarefa
exports.criarTarefa = async (req, res) => {
  const { nome, descricao } = req.body;

  const query = 'INSERT INTO tarefas (nome, descricao) VALUES ($1, $2) RETURNING *';
  const values = [nome, descricao];

  try {
    const result = await pool.query(query, values);
    const tarefa = result.rows[0];
    res.status(201).json(tarefa);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

// Listar todas as tarefas
exports.listarTarefas = async (req, res) => {
  const query = 'SELECT * FROM tarefas';

  try {
    const result = await pool.query(query);
    res.status(200).json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

// Editar uma tarefa
exports.editarTarefa = async (req, res) => {
  const { id } = req.params;
  const { nome, descricao, status } = req.body;

  const query = `
    UPDATE tarefas SET nome = $1, descricao = $2, status = $3, updated_at = CURRENT_TIMESTAMP
    WHERE id = $4 RETURNING *`;
  const values = [nome, descricao, status, id];

  try {
    const result = await pool.query(query, values);
    if (result.rows.length === 0) {
      return res.status(404).json({ message: 'Tarefa não encontrada' });
    }
    res.status(200).json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

// Excluir uma tarefa
exports.excluirTarefa = async (req, res) => {
  const { id } = req.params;

  const query = 'DELETE FROM tarefas WHERE id = $1 RETURNING *';
  const values = [id];

  try {
    const result = await pool.query(query, values);
    if (result.rows.length === 0) {
      return res.status(404).json({ message: 'Tarefa não encontrada' });
    }
    res.status(200).json({ message: 'Tarefa excluída com sucesso' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};
```

#### Passo 2 — Definindo as Rotas

Agora, crie as rotas para interagir com as funções de **CRUD** de tarefas. Crie o arquivo `routes/index.js`:

```javascript
// routes/index.js
const express = require('express');
const router = express.Router();
const TarefaController = require('../controllers/TarefaController');

// Rotas para o CRUD de tarefas
router.post('/tarefas', TarefaController.criarTarefa);
router.get('/tarefas', TarefaController.listarTarefas);
router.put('/tarefas/:id', TarefaController.editarTarefa);
router.delete('/tarefas/:id', TarefaController.excluirTarefa);

module.exports = router;
```

#### Passo 3 — Configurando o Servidor Express

Edite o arquivo `server.js` na raiz do projeto para configurar o servidor Express:

```javascript
// server.js
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const routes = require('./routes');

const app = express();
const port = 3000;

// Middlewares
app.use(cors());
app.use(bodyParser.json());

// Usando as rotas definidas
app.use('/api', routes);

app.listen(port, () => {
  console.log(`Servidor rodando na porta ${port}`);
});
```

#### Passo 4 — Testando o Sistema

Agora, você pode rodar o servidor com:

```bash
node server.js
```

E testar os endpoints do CRUD (usando o Postman ou outra ferramenta similar):

1. **Criar uma tarefa** (POST `/api/tarefas`)
2. **Listar todas as tarefas** (GET `/api/tarefas`)
3. **Editar uma tarefa** (PUT `/api/tarefas/:id`)
4. **Excluir uma tarefa** (DELETE `/api/tarefas/:id`)

### Etapa 3 — Arquitetura MVC
A Arquitetura MVC (Model-View-Controller) é uma abordagem que organiza a aplicação em três componentes principais:

- `Model`: Representa a estrutura dos dados e interage diretamente com o banco de dados. No seu projeto, a modelagem dos dados é feita por meio de comandos SQL, sem o uso de ORM, onde a estrutura das tabelas é criada e gerenciada diretamente com SQL.

- `View`: Responsável pela interface com o usuário. Neste projeto, a view pode ser gerada por um frontend que consome os endpoints da API (ou pode ser algo mais simples, como um cliente REST que envia requisições).

- `Controller`: Gerencia a lógica de negócios e atua como intermediário entre o Model e a View. Os controllers recebem as requisições HTTP, validam os dados e executam a manipulação do banco de dados, respondendo de forma adequada com os resultados.

- Exemplo de MVC na Prática:
    - Model: A tabela tarefas no banco de dados.

    - Controller: Funções como criarTarefa, listarTarefas, editarTarefa, e excluirTarefa que manipulam as informações da tabela.

    - View: O frontend da aplicação ou uma API que consome as rotas para exibir ou interagir com os dados.

#### Passo 1 - Criando o diagrama de arquitetura
Para criar o diagrama de arquitetura MVC (Model-View-Controller), siga estas etapas:

- `Model`: Desenhe uma caixa que representa a camada de dados. Conecte-a às tabelas do banco de dados, demonstrando que o Model interage diretamente com o banco.

- `View`: Crie uma caixa que representa a interface de usuário. Ela interage com o Controller para exibir as informações para o usuário.

- `Controller`: Adicione uma caixa para o Controller, conectando-a tanto ao Model quanto à View, indicando que ele manipula a lógica de negócios e lida com a comunicação entre a interface e os dados.

- Relacionamentos: Use setas para conectar os componentes, demonstrando como os dados fluem entre o Model, Controller e View.

Use este exemplo como referência: https://github.com/kterra/Inteli-2024-1B/blob/main/materiais/ponderada-2/exemplo-arq-v2.jpg

### Etapa 4 — Atulizando a documentação

Adicione a documentação sobre como configurar o banco de dados, como rodar as migrações e como testar as APIs. 

---

##  Requisitos Mínimos da Entrega
- Criação do banco de dados PostgreSQL e conexão com o Banco de Dados
- Migração funcional
- Implementação de Models
- Implementação de Controllers
- Rotas Funcionando
- Arquitetura MVC Completa
- Documentação Completa
