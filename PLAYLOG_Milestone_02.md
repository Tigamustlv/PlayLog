Milestone 02 — Autenticação, Catálogo de Jogos e Biblioteca

Status: CONCLUÍDO

Objetivo

O primeiro milestone do Playlog teve como objetivo construir a base funcional da API, implementando autenticação de usuários, gerenciamento de perfil, catálogo de jogos e biblioteca pessoal.

Neste estágio, a aplicação utiliza arquivos JSON como mecanismo temporário de persistência, permitindo desenvolver e validar as regras de negócio antes da implementação de um banco de dados definitivo.

1. Autenticação e usuários

Foi implementado o fluxo completo de autenticação de usuários.

Registro

Foi criado o endpoint POST /auth/register, responsável pelo cadastro de novos usuários.

O cadastro possui validações para impedir a utilização de um email ou username que já esteja cadastrado.

As senhas não são armazenadas diretamente. Antes da persistência, elas são transformadas em hash utilizando BCrypt através do Passlib.

O usuário também possui informações de perfil, como username, avatar, biografia e visibilidade do perfil.

Login

Foi criado o endpoint POST /auth/login.

O processo de login valida o email e a senha informados pelo usuário. Após a autenticação, a API gera um JSON Web Token (JWT).

O token possui um período de expiração e utiliza o email do usuário como identificador no campo sub.

A chave utilizada para assinar o token é armazenada através de variável de ambiente, evitando deixar a chave diretamente no código.

Usuário autenticado

Foi implementada a dependência obter_usuario_atual, responsável por validar o JWT recebido no header Authorization.

O fluxo é:

JWT → validação → identificação do usuário → usuário autenticado.

Essa dependência é utilizada nos endpoints protegidos e impede que o cliente precise enviar manualmente o identificador do próprio usuário.

2. Perfil do usuário

Foi implementado o endpoint GET /users/me, responsável por retornar os dados do usuário autenticado.

Também foi implementado PATCH /users/me, permitindo atualizar informações do próprio perfil.

Os campos disponíveis para atualização são username, avatar, biografia e visibilidade do perfil.

O username possui uma validação de unicidade, impedindo que dois usuários possuam o mesmo nome.

A utilização de /me garante que o usuário trabalhe sempre sobre o próprio perfil, utilizando sua identidade obtida através do JWT.

3. Catálogo de jogos

Foi criado o catálogo global de jogos através do arquivo games.json.

Cada jogo possui informações próprias, como:

ID
Nome
Descrição
URL da capa
Data de lançamento
Cadastro de jogos

Foi implementado POST /games.

O endpoint permite cadastrar novos jogos e possui validações para impedir que o mesmo jogo seja cadastrado mais de uma vez.

A comparação do nome ignora diferenças entre letras maiúsculas e minúsculas.

Listagem e pesquisa

Foi implementado GET /games.

O endpoint permite listar todos os jogos cadastrados e também realizar uma pesquisa simples utilizando o parâmetro search.

Exemplo conceitual: GET /games?search=elden.

A pesquisa é feita pelo nome do jogo e não diferencia letras maiúsculas e minúsculas.

Quando uma pesquisa não encontra resultados, a API retorna uma lista vazia. Isso diferencia uma pesquisa sem resultados da consulta de um recurso específico que não existe.

Consulta individual

Foi implementado GET /games/{game_id} para consultar um jogo específico através do seu ID.

Quando o jogo não existe, a API retorna 404 Not Found.

4. Plataformas

Foi criado o arquivo platforms.json para representar as plataformas disponíveis no sistema.

As plataformas possuem um identificador e um nome.

O platform_id é utilizado internamente para criar relações entre os jogos da biblioteca e suas respectivas plataformas.

A plataforma não é armazenada como uma cópia dentro do jogo. A relação é feita através do identificador.

Na apresentação da biblioteca, o nome da plataforma é retornado ao cliente em vez de apenas seu ID.

5. Biblioteca do usuário

Foi implementado o conceito de LibraryGame, responsável por representar a relação entre um usuário, um jogo e uma plataforma.

A estrutura lógica é:

Usuário → LibraryGame → Game
　　　　　　　　　└→ Platform

O LibraryGame também possui o status do jogo.

A biblioteca é persistida em arquivo JSON durante este estágio do projeto.

Adicionar jogo

Foi implementado POST /users/me/library/games.

O endpoint recebe o jogo, a plataforma e o status desejado.

Antes de criar a relação, o backend:

Identifica o usuário através do JWT.
Verifica se o jogo existe no catálogo.
Verifica se a plataforma existe.
Verifica se aquela combinação já está na biblioteca.
Cria o LibraryGame.

Um mesmo jogo pode ser adicionado mais de uma vez à biblioteca desde que esteja associado a plataformas diferentes.

Por exemplo, um usuário pode possuir Elden Ring no PC e no PlayStation 5.

A combinação considerada duplicada é:

Usuário + Jogo + Plataforma.

Portanto, o mesmo jogo na mesma plataforma não pode ser adicionado duas vezes.

6. Status dos jogos

Foi definido que todo jogo dentro da biblioteca possui um status.

Os estados disponíveis são:

a iniciar
jogando
zerado
dropado

A validação é realizada através do Pydantic utilizando Literal, impedindo que valores fora dos estados definidos sejam aceitos pela API.

O status pertence ao LibraryGame, e não ao jogo global.

Isso é importante porque o mesmo jogo pode possuir status diferentes em plataformas diferentes.

Por exemplo:

Elden Ring → PC → jogando

Elden Ring → PlayStation 5 → zerado

O jogo no catálogo continua sendo o mesmo, enquanto cada relação na biblioteca possui seu próprio estado.

7. Consulta da biblioteca

Foi implementado GET /users/me/library.

O endpoint utiliza o JWT para identificar o usuário autenticado e retorna somente os jogos pertencentes à biblioteca daquele usuário.

O backend realiza o relacionamento entre os dados armazenados:

JWT → Usuário → LibraryGame → Game + Platform.

O game_id é utilizado para recuperar os dados do jogo no catálogo.

O platform_id é utilizado para recuperar o nome da plataforma.

Dessa forma, a resposta apresenta informações úteis para o cliente sem precisar expor apenas os identificadores internos.

A biblioteca também permite que o mesmo jogo apareça mais de uma vez quando estiver associado a plataformas diferentes.

8. Atualização do jogo na biblioteca

Foi implementado PATCH /users/me/library/games/{game_id}.

Esse endpoint foi projetado para alterar somente informações relacionadas ao LibraryGame.

Atualmente, a única informação que pode ser modificada é o status.

O platform_id é informado para identificar qual associação do jogo deve ser alterada.

Isso é necessário porque o mesmo jogo pode estar presente em mais de uma plataforma e cada plataforma pode possuir um status diferente.

O endpoint não permite alterar informações do jogo global, como nome, descrição, capa ou data de lançamento.

Essas informações pertencem ao catálogo de jogos e não à biblioteca individual do usuário.

9. Remoção da biblioteca

Foi implementado DELETE /users/me/library/games/{game_id}.

O endpoint identifica o usuário através do JWT e verifica se existe uma relação entre aquele usuário, o jogo e a plataforma informada.

Quando a relação existe, somente o LibraryGame é removido.

O jogo original do catálogo não é excluído.

Essa separação garante que remover um jogo da biblioteca de um usuário não afete o catálogo global do Playlog.

Assim como na atualização, o platform_id é considerado para permitir que um usuário remova especificamente a versão do jogo associada a determinada plataforma.

10. Segurança e controle de acesso

Os endpoints relacionados ao usuário e à biblioteca utilizam autenticação através de JWT.

O usuário não informa diretamente qual biblioteca deseja acessar.

O backend identifica o usuário através do token enviado na requisição.

Isso permite garantir que:

Um usuário só consulte a própria biblioteca.
Um usuário só adicione jogos à própria biblioteca.
Um usuário só altere seus próprios LibraryGames.
Um usuário só remova seus próprios LibraryGames.

O identificador do usuário não é confiado a partir dos dados enviados pelo cliente; ele é obtido através da autenticação.

11. Persistência atual

Neste milestone, a persistência foi implementada utilizando arquivos JSON.

Os principais arquivos são:

users.json — usuários e informações de autenticação/perfil.
games.json — catálogo global de jogos.
platforms.json — plataformas disponíveis.
Arquivo da biblioteca — relações entre usuários, jogos e plataformas.

Essa solução é temporária e foi utilizada para permitir o desenvolvimento inicial da aplicação sem introduzir imediatamente a complexidade de um banco de dados.

A arquitetura desenvolvida permite que essa camada seja posteriormente substituída por um banco de dados real.

12. Endpoints concluídos

Ao final deste milestone, os seguintes endpoints foram implementados:

POST /auth/register — cadastro de usuário.
POST /auth/login — autenticação e geração de JWT.
GET /users/me — consulta do próprio perfil.
PATCH /users/me — atualização do próprio perfil.
POST /games — cadastro de jogos.
GET /games — listagem e pesquisa de jogos.
GET /games/{game_id} — consulta de jogo específico.
POST /users/me/library/games — adição de jogo à biblioteca.
GET /users/me/library — consulta da biblioteca.
PATCH /users/me/library/games/{game_id} — alteração do status de um jogo na biblioteca.
DELETE /users/me/library/games/{game_id} — remoção de um jogo da biblioteca.
13. Regras de negócio consolidadas

Ao final do milestone, as principais regras implementadas são:

Usuários possuem email único.
Usuários possuem username único.
Senhas são armazenadas como hash.
A autenticação utiliza JWT.
O usuário autenticado é identificado através do token.
Jogos possuem cadastro único no catálogo.
Jogos podem ser pesquisados pelo nome.
Plataformas são entidades independentes.
Um usuário pode possuir o mesmo jogo em diferentes plataformas.
A combinação Usuário + Jogo + Plataforma não pode ser duplicada.
Cada relação de biblioteca possui seu próprio status.
O status pode ser a iniciar, jogando, zerado ou dropado.
O status é independente para cada plataforma.
Alterações na biblioteca não modificam o jogo do catálogo.
Remover um jogo da biblioteca não remove o jogo do catálogo.
Usuários só podem manipular suas próprias bibliotecas.
Conclusão

O Milestone 01 foi concluído com a implementação da base funcional do Playlog.

A aplicação agora possui autenticação, autorização, gerenciamento de usuários, catálogo de jogos, plataformas e biblioteca pessoal, incluindo o relacionamento entre usuário, jogo e plataforma.

A principal estrutura de domínio estabelecida neste milestone é a separação entre o Game, que representa o jogo no catálogo global, e o LibraryGame, que representa a posse daquele jogo por um usuário em uma plataforma específica e mantém seu status individual.

Essa estrutura permite que o sistema evolua posteriormente para funcionalidades mais complexas sem precisar alterar o conceito fundamental da biblioteca.

Status final: Milestone 02 concluído.