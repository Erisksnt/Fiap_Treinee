# Análise da aplicação ToggleMaster

## 1. Visão geral

O ToggleMaster é uma aplicação monolítica que oferece uma API HTTP para criar, consultar e alterar *feature flags*. Uma flag representa uma funcionalidade identificada por um nome e pode estar habilitada (`true`) ou desabilitada (`false`).

O sistema tem dois componentes principais:

- **Aplicação web:** escrita em Python com Flask e executada pelo servidor Gunicorn.
- **Banco de dados:** PostgreSQL, que armazena as flags.

No ambiente local, o Docker Compose executa cada componente em um container. A aplicação se conecta ao banco pela rede interna do Compose.

## 2. Funcionamento de `app.py`

O arquivo `app.py` define a aplicação Flask, os endpoints da API e o acesso ao PostgreSQL por meio de `psycopg2`.

### Configuração e conexão com o banco

As configurações de conexão são lidas das variáveis de ambiente `DB_HOST`, `DB_NAME`, `DB_USER` e `DB_PASSWORD`. A função `get_db_connection()` usa esses valores para abrir uma conexão com o PostgreSQL.

Usar variáveis de ambiente permite configurar a aplicação para diferentes ambientes sem colocar os parâmetros de conexão diretamente no código. As credenciais declaradas no Compose são apenas valores de desenvolvimento e devem ser substituídas por configurações apropriadas em um ambiente real.

### Tabela `flags`

O comando Flask `init-db` chama `init_db()`, que cria a tabela `flags` caso ela ainda não exista. A tabela contém:

| Coluna | Finalidade |
| --- | --- |
| `id` | Identificador numérico gerado pelo banco e chave primária. |
| `name` | Nome da flag; obrigatório e único. |
| `is_enabled` | Estado booleano, desabilitado por padrão. |
| `created_at` | Data e hora da criação, preenchidas pelo banco. |

### Endpoints

| Método e rota | Função |
| --- | --- |
| `GET /health` | Retorna `{"status":"ok"}` para indicar que o processo web responde. Não verifica, por si só, se o banco está acessível. |
| `POST /flags` | Cria uma flag. Recebe `name` e, opcionalmente, `is_enabled` (padrão `false`). Responde `201` em caso de sucesso, `400` se faltar `name` e `409` se o nome já existir. |
| `GET /flags` | Lista as flags em ordem alfabética pelo nome. |
| `GET /flags/<name>` | Busca uma flag pelo nome. Responde `404` quando ela não existe. |
| `PUT /flags/<name>` | Atualiza o estado da flag. Exige `is_enabled` como valor booleano; responde `400` se o campo for inválido e `404` se a flag não existir. |

As operações de leitura e escrita abrem uma conexão com o banco, executam a consulta e fecham cursor e conexão. Os valores recebidos nas consultas SQL são passados como parâmetros ao `psycopg2`, em vez de serem concatenados no texto SQL.

## 3. Como o `Dockerfile` monta a aplicação

O `Dockerfile` define a imagem do container da aplicação:

1. Parte da imagem `python:3.9-slim`.
2. Define `/app` como diretório de trabalho.
3. Copia `requirements.txt` e instala Flask, Werkzeug, `psycopg2-binary` e Gunicorn.
4. Copia o script `entrypoint.sh` e o `app.py`.
5. Instala o cliente PostgreSQL, que fornece `pg_isready` para verificar se o banco aceita conexões.
6. Declara a porta `5000` e configura o script de entrada do container.

O Gunicorn serve a aplicação Flask na porta `5000`, escutando em `0.0.0.0` para aceitar conexões externas ao container. A linha `CMD` também declara o comando Gunicorn, mas o `entrypoint.sh` inicia o Gunicorn diretamente.

## 4. Como o Docker Compose coordena os serviços

O arquivo `docker-compose.yaml` define dois serviços:

- **`app`:** constrói a imagem a partir do `Dockerfile`, publica a porta `5000` da máquina para a porta `5000` do container e recebe as configurações de conexão com o banco. O endereço `DB_HOST=db` usa o nome do serviço `db` como endereço na rede interna do Compose. O volume `.:/app` monta os arquivos locais no container, facilitando o desenvolvimento.
- **`db`:** usa a imagem `postgres:13` e inicializa o banco `togglemaster` com o usuário e a senha definidos no Compose. O volume nomeado `postgres_data` mantém os dados do PostgreSQL mesmo quando os containers são removidos.

O serviço `db` tem um *health check* baseado em `pg_isready`. A aplicação depende de `db` estar saudável antes de iniciar. Isso ajuda a evitar uma tentativa de conexão antes de o PostgreSQL estar pronto, mas não substitui o tratamento de erros de conexão durante a execução.

## 5. Papel do `entrypoint.sh`

Antes de iniciar a API, o script:

1. Confere se as principais variáveis de ambiente do banco estão definidas.
2. Aguarda o PostgreSQL responder a `pg_isready`.
3. Executa `flask init-db` para criar a tabela.
4. Inicia o Gunicorn na porta `5000`.

O script deve usar finais de linha `LF`, compatíveis com Linux, pois é executado dentro do container. Finais de linha `CRLF` podem fazer o shell interpretar incorretamente comandos do arquivo.

**Ponto de atenção:** o script usa `flask init-db || true`, que permite continuar mesmo se a inicialização da tabela falhar. Além disso, `init_db()` captura exceções e não as propaga. Assim, uma falha ao criar a tabela pode não impedir o Gunicorn de iniciar; as rotas que acessam o banco podem então retornar erro. Para produção, é importante tornar falhas de inicialização visíveis e impedir que o serviço seja considerado pronto quando a configuração essencial falhar.

## 6. Fluxo completo de inicialização

```text
docker compose up
        |
        +--> PostgreSQL inicia e passa no health check
        |
        +--> Container da aplicação inicia
                 |
                 +--> entrypoint.sh aguarda o banco
                 +--> flask init-db cria a tabela flags
                 +--> Gunicorn serve app:app na porta 5000
                          |
                          +--> API disponível em http://localhost:5000
```

Para validar localmente, `GET http://localhost:5000/health` deve responder com o estado `ok`. As rotas de flags também dependem de o banco estar acessível e da tabela ter sido criada.

## 7. Características da arquitetura e considerações

Como monólito, a API e suas regras de negócio estão reunidas em uma única aplicação, implantada como um serviço. Isso simplifica a execução e é adequado ao escopo inicial do desafio, embora a aplicação e o banco continuem sendo serviços separados.

Para o deploy na AWS descrito no README, a aplicação pode rodar na EC2 e conectar-se a um PostgreSQL no RDS. Nesse cenário, `DB_HOST`, `DB_NAME`, `DB_USER` e `DB_PASSWORD` devem apontar para o RDS, e as regras de rede devem permitir a conexão da EC2 à porta `5432` do banco. O Compose local é uma configuração de desenvolvimento; os valores de usuário e senha nele não devem ser tratados como credenciais de produção.

## 8. Vantagens e Desvantagens do Monólito

No ToggleMaster, a API e suas operações sobre *feature flags* fazem parte de uma única aplicação Flask, executada como um serviço web. O PostgreSQL é um serviço separado, não um componente embutido no processo da aplicação. Para o escopo atual, essa organização oferece vantagens práticas:

### Vantagens

- **Simplicidade de desenvolvimento e deploy:** há uma aplicação para configurar, executar e publicar; o Docker Compose coordena a API e o banco no ambiente local.
- **Menor complexidade operacional:** não é necessário manter comunicação entre vários serviços de aplicação, descoberta de serviços ou pipelines de deploy independentes.
- **Alterações mais diretas:** endpoints e regras relacionadas podem ser desenvolvidos e testados no mesmo projeto, sem coordenar versões de múltiplos serviços.
- **Adequação ao escopo:** para uma API pequena, com responsabilidades relacionadas, a estrutura evita a complexidade prematura de dividir o sistema.
- **Escala inicial independente do banco:** embora a aplicação seja monolítica, o serviço web e o PostgreSQL são separados e podem receber recursos de infraestrutura distintos.
- **Projeto MVP:** um MVP que atende a necessidade inicial de um projeto.

### Desvantagens

- **Escala menos granular:** se apenas uma parte da aplicação exigir mais capacidade, normalmente será necessário escalar instâncias da aplicação inteira. O Compose atual não configura escalabilidade ou balanceamento para produção.
- **Deploy acoplado:** mudanças em qualquer parte da aplicação exigem a publicação do mesmo artefato, mesmo que outras partes não tenham mudado.
- **Falhas e recursos compartilhados:** problemas de consumo de CPU, memória ou bloqueio no processo da aplicação podem afetar todos os endpoints servidos por ele.
- **Acoplamento tende a crescer:** sem separação clara de responsabilidades e testes, novas funcionalidades podem tornar o código mais difícil de modificar e manter.
- **Evolução tecnológica menos independente:** a aplicação compartilha linguagem, processo de execução e ciclo de publicação; a substituição isolada de uma tecnologia pode exigir mais esforço.

Essas desvantagens não tornam o monólito inadequado por si só. Para o tamanho atual do ToggleMaster, manter a aplicação como um monólito é uma escolha razoável; uma separação em serviços só se justificaria diante de necessidades concretas de escala, autonomia de equipes ou isolamento operacional.

## 9. Análise baseada nos 12-Factor App

| Fator | Avaliação do ToggleMaster |
| --- | --- |
| **1. Base de código (Codebase)** | O projeto mantém a aplicação em uma base de código, adequada para uma instância de serviço. O documento não detalha a relação entre repositório e ambientes implantados. |
| **2. Dependências (Dependencies)** | **Atendido em boa parte:** as dependências Python estão declaradas em `requirements.txt` e instaladas na imagem. O arquivo fixa versões das bibliotecas Python, mas a imagem base e o pacote de sistema do cliente PostgreSQL não estão fixados por digest ou versão específica. |
| **3. Configuração (Config)** | **Parcialmente atendido:** os dados de conexão são lidos de variáveis de ambiente. No Compose, porém, usuário e senha estão definidos como valores de desenvolvimento; não devem ser reutilizados em produção. Outras configurações, como a porta web, permanecem no código ou na imagem. |
| **4. Serviços de apoio (Backing services)** | **Atendido:** o PostgreSQL é acessado como um serviço separado. No Compose ele é outro container; em produção, a configuração pode apontar para um serviço gerenciado, como o RDS, sem mudar a função principal da API. |
| **5. Build, release e execução (Build, release, run)** | **Parcialmente atendido:** o Dockerfile constrói a imagem e o Compose executa os containers, mas o projeto não descreve releases imutáveis ou um pipeline que separe claramente build e release. A inicialização da tabela ocorre no `entrypoint.sh` antes de iniciar o servidor. |
| **6. Processos (Processes)** | **Atendido para a aplicação web:** o estado persistente das flags fica no PostgreSQL, não na memória do processo Flask. O volume nomeado do Compose preserva os dados do banco. |
| **7. Associação de portas (Port binding)** | **Atendido:** o Gunicorn escuta em `0.0.0.0:5000`, e o Compose publica a porta `5000` do container. |
| **8. Concorrência (Concurrency)** | **Possível, mas não demonstrado na configuração atual:** processos ou instâncias web adicionais podem ser executados para atender mais requisições, mantendo os dados no banco compartilhado. O Compose apresentado não configura essa escala horizontal. |
| **9. Descartabilidade (Disposability)** | **Parcialmente atendido:** o `entrypoint.sh` usa `exec` para iniciar o Gunicorn, permitindo que ele receba sinais do container, e aguarda o banco antes de iniciar. A espera não tem limite de tempo e a inicialização da tabela pode falhar sem impedir o início do servidor, devido ao `|| true` e ao tratamento de exceções em `init_db()`. |
| **10. Paridade entre desenvolvimento e produção (Dev/prod parity)** | **Não é possível confirmar:** o Compose fornece um ambiente local containerizado, enquanto o deploy descrito pode usar EC2 e RDS. Não há informações suficientes sobre frequência de deploy, diferenças de versões ou semelhança entre os ambientes. |
| **11. Logs (Logs)** | **Parcialmente atendido:** mensagens da aplicação são escritas na saída padrão e o Gunicorn também pode emitir logs para os fluxos padrão, que são capturáveis pelo runtime de containers. O projeto não configura agregação, retenção ou monitoramento centralizado. |
| **12. Processos administrativos (Admin processes)** | **Parcialmente atendido:** existe o comando Flask `init-db` para preparar a tabela, mas ele é executado automaticamente na inicialização da aplicação. O projeto não documenta um fluxo separado e controlado para tarefas administrativas ou migrações em produção. |

Em resumo, o projeto já adota práticas importantes, como dependências declaradas, configuração de conexão por ambiente, serviços separados e execução web vinculada a uma porta. Os principais pontos a aprimorar para uma operação mais alinhada aos 12 fatores são externalizar toda a configuração sensível, tornar releases e tarefas administrativas explícitos, falhar de forma visível quando a inicialização essencial do banco não funcionar e definir estratégias de escala, logs e paridade de ambientes para produção.
