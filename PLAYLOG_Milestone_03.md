Milestone 03 — Experiência do Jogo

Status: CONCLUÍDO

Objetivo

O terceiro milestone do Playlog teve como objetivo evoluir a biblioteca de jogos para representar não apenas quais jogos pertencem ao usuário, mas também qual foi sua experiência individual com cada jogo.

Neste estágio, foram implementadas avaliações, reviews, consulta da experiência individual do usuário e acompanhamento de progresso e tempo jogado.

A estrutura continua utilizando arquivos JSON como mecanismo temporário de persistência, mantendo a arquitetura desenvolvida nos milestones anteriores.

1. Avaliação de jogos

Foi implementado o sistema de avaliações associado aos jogos presentes na biblioteca do usuário.

A avaliação é vinculada à combinação:

Usuário + Jogo + Plataforma

Isso permite que a experiência de um usuário seja registrada individualmente para cada versão de um jogo presente em sua biblioteca.

Por exemplo:

Usuário
├── Elden Ring → PC
│   └── rating: 9
│
└── Elden Ring → PlayStation 5
    └── rating: 10


A avaliação não pertence diretamente ao Game global.

Ela representa a experiência daquele usuário com aquela associação específica da biblioteca.

2. Criação de review

Foi implementado:

POST /users/me/library/games/{game_id}/review

O endpoint recebe:

platform_id
rating
review

Exemplo conceitual:

{
    "platform_id": 2,
    "rating": 9,
    "review": "Zerei e curti mais, jogo muito bom"
}


O backend:

identifica o usuário através do JWT;
verifica se o jogo está na biblioteca do usuário;
verifica a plataforma associada;
valida a nota;
verifica se já existe uma avaliação para aquela relação;
cria a avaliação quando ela ainda não existe;
atualiza a avaliação quando ela já existe.

A nota possui validação entre 0 e 10.

A review possui validação de tamanho, aceitando entre 1 e 2000 caracteres.

As avaliações são persistidas temporariamente em reviews.json.

3. Consulta das avaliações de um jogo

Foi implementado:

GET /games/{game_id}/reviews

O endpoint retorna as avaliações existentes para determinado jogo.

Além das avaliações individuais, a API calcula:

média das avaliações;
quantidade total de avaliações.

Exemplo conceitual:

{
    "game_id": 1,
    "average_rating": 8,
    "total_reviews": 1,
    "reviews": [
        {
            "id": 1,
            "user_email": "usuario@email.com",
            "game_id": 1,
            "platform_id": 4,
            "rating": 8,
            "review": "Muito bom."
        }
    ]
}


A média é calculada a partir das avaliações existentes.

O sistema passa a possuir, portanto, a relação:

Game
├── média das avaliações
└── avaliações dos usuários

4. Experiência individual do usuário

Foi implementado:

GET /users/me/library/games/{game_id}

O endpoint representa a experiência daquele usuário com um jogo específico dentro de sua biblioteca.

Como um mesmo jogo pode existir em diferentes plataformas, o platform_id é utilizado para identificar a associação correta.

Exemplo:

GET /users/me/library/games/2?platform_id=2

A resposta combina informações de diferentes arquivos:

JWT
 ↓
Usuário
 ↓
LibraryGame
 ├── Game
 ├── Platform
 └── Status
       ↓
Review


A resposta pode conter:

{
    "game": {
        "id": 2,
        "name": "Red Dead Redemption 2",
        "description": "Aventura de ação em mundo aberto ambientada no Velho Oeste americano.",
        "cover_url": "https://example.com/rdr2.jpg",
        "release_date": "2018-10-26"
    },
    "platform": {
        "id": 2,
        "name": "PlayStation 5"
    },
    "status": "zerado",
    "rating": 10,
    "review": "GOTY"
}


Quando o usuário ainda não possui uma avaliação para aquele jogo e plataforma, rating e review são retornados como null.

Essa separação diferencia:

GET /games/{game_id}

de:

GET /users/me/library/games/{game_id}

O primeiro representa as informações globais do jogo.

O segundo representa a relação e a experiência do usuário com aquele jogo.

5. Atualização de review

Foi implementado:

PATCH /users/me/library/games/{game_id}/review

O endpoint permite alterar:

rating
review

A identificação da avaliação utiliza:

Usuário + Jogo + Plataforma


O backend verifica se:

o jogo pertence à biblioteca do usuário;
a avaliação existe;
a avaliação pertence ao usuário autenticado.

Quando a avaliação existe, somente os dados da avaliação são alterados.

Informações como:

jogo;
plataforma;
status;

não são modificadas por esse endpoint.

Exemplo:

{
    "platform_id": 2,
    "rating": 9,
    "review": "Zerei e curti mais, jogo muito bom"
}

6. Remoção de review

Foi implementado:

DELETE /users/me/library/games/{game_id}/review

O platform_id é utilizado para identificar a avaliação correspondente.

Quando a avaliação existe, ela é removida do reviews.json.

A remoção da avaliação não remove o jogo da biblioteca.

Após a exclusão, o LibraryGame continua existindo normalmente.

A experiência passa a apresentar:

{
    "status": "zerado",
    "rating": null,
    "review": null
}


Dessa forma, a avaliação é independente da existência do jogo na biblioteca.

7. Progresso do jogo

Foi implementado o acompanhamento manual do progresso do jogo através do LibraryGame.

Foram adicionados os campos:

minutes_played
progress


A estrutura do LibraryGame passa a ser conceitualmente:

LibraryGame
├── user
├── game
├── platform
├── status
├── minutes_played
└── progress


Ao adicionar um novo jogo à biblioteca, os valores iniciais são:

minutes_played = 0
progress = 0

8. Tempo jogado

O tempo de jogo é armazenado através do campo:

minutes_played

O campo utiliza valores inteiros.

Exemplos:

30 minutos → 30
40 minutos → 40
1 hora → 60
1h30 → 90
42h15 → 2535


A decisão de armazenar o tempo em minutos foi tomada considerando uma futura integração com APIs externas, especialmente a Steam.

Dessa forma, o Playlog evita perda de precisão que poderia ocorrer ao armazenar diretamente horas como números inteiros.

Também evita a necessidade de armazenar valores decimais como:

0.6667 horas


para representar apenas 40 minutos.

A conversão para horas poderá ser realizada posteriormente na camada de apresentação.

9. Progresso

O campo progress representa o percentual de progresso do usuário no jogo.

O valor permitido está entre:

0 e 100


Exemplos:

0   → início
25  → 25%
50  → 50%
75  → 75%
100 → concluído


O progresso é mantido separado do status.

Isso significa que alterar o progresso não altera automaticamente o status do jogo.

Por exemplo:

progress = 50
status = "a iniciar"


continua sendo tecnicamente possível neste estágio.

As regras para possíveis transições automáticas de status poderão ser definidas futuramente.

10. Atualização do progresso

Foi implementado:

PATCH /users/me/library/games/{game_id}/progress

O endpoint recebe:

platform_id
minutes_played
progress

Exemplo:

{
    "platform_id": 2,
    "minutes_played": 40,
    "progress": 25
}


O backend identifica o usuário através do JWT e localiza o LibraryGame correspondente.

Somente os campos de progresso são alterados.

O status do jogo permanece independente.

11. Validação do progresso

Foram implementadas validações utilizando Pydantic.

minutes_played não pode possuir valor negativo.

minutes_played >= 0


progress deve estar entre 0 e 100.

0 <= progress <= 100


Valores inválidos são rejeitados pela API através de:

422 Unprocessable Entity

Exemplos de valores inválidos:

{
    "platform_id": 2,
    "minutes_played": -10,
    "progress": 50
}


ou:

{
    "platform_id": 2,
    "minutes_played": 100,
    "progress": 101
}


Essa validação impede que dados inconsistentes sejam armazenados.

12. Separação entre Game, LibraryGame e Review

Com a evolução do domínio, a separação entre as entidades tornou-se mais importante.

Game

Representa o jogo global no catálogo.

Game
├── nome
├── descrição
├── capa
└── data de lançamento

LibraryGame

Representa a relação de um usuário com um jogo em determinada plataforma.

LibraryGame
├── usuário
├── jogo
├── plataforma
├── status
├── minutes_played
└── progress

Review

Representa a avaliação do usuário sobre aquela relação.

Review
├── usuário
├── jogo
├── plataforma
├── rating
└── review


Essa separação permite que o mesmo jogo possua diferentes experiências dependendo do usuário e da plataforma.

13. Segurança e controle de acesso

Os endpoints relacionados à experiência do usuário utilizam autenticação através de JWT.

O usuário é identificado através do token utilizando a dependência:

obter_usuario_atual

O backend não confia em um identificador de usuário enviado diretamente pelo cliente.

As operações de biblioteca e avaliação utilizam a identidade obtida através do JWT.

Isso garante que:

um usuário só altere seu próprio progresso;
um usuário só consulte sua própria relação na biblioteca;
um usuário só crie avaliações para jogos presentes em sua biblioteca;
um usuário só altere suas próprias avaliações;
um usuário só remova suas próprias avaliações.
14. Persistência atual

A persistência continua sendo realizada através de arquivos JSON.

Os principais arquivos envolvidos no milestone são:

games.json
platforms.json
libraries.json
reviews.json


A estrutura continua sendo temporária e tem como objetivo permitir o desenvolvimento das regras de negócio antes da migração para um banco de dados definitivo.

A criação de reviews.json adiciona uma nova camada de persistência responsável pelas avaliações dos usuários.

15. Endpoints concluídos

Ao final deste milestone, os seguintes endpoints foram implementados:

Reviews

POST /users/me/library/games/{game_id}/review

Criação ou atualização de uma avaliação.

GET /games/{game_id}/reviews

Consulta das avaliações de um jogo e cálculo da média.

PATCH /users/me/library/games/{game_id}/review

Atualização da própria avaliação.

DELETE /users/me/library/games/{game_id}/review

Remoção da própria avaliação.

Experiência na biblioteca

GET /users/me/library/games/{game_id}

Consulta dos dados globais do jogo combinados com a experiência do usuário.

Progresso

PATCH /users/me/library/games/{game_id}/progress

Atualização do tempo jogado e do percentual de progresso.

16. Regras de negócio consolidadas

Ao final do milestone, as principais regras implementadas são:

Avaliações pertencem à relação entre usuário, jogo e plataforma.
Um usuário só pode avaliar um jogo que esteja em sua biblioteca.
A mesma relação usuário + jogo + plataforma não possui múltiplas avaliações.
A nota possui valor entre 0 e 10.
A review possui entre 1 e 2000 caracteres.
O usuário só pode alterar sua própria avaliação.
O usuário só pode remover sua própria avaliação.
Remover uma avaliação não remove o jogo da biblioteca.
O jogo global não é alterado pelas avaliações.
O status continua pertencendo ao LibraryGame.
O progresso pertence ao LibraryGame.
minutes_played não pode ser negativo.
progress deve estar entre 0 e 100.
O progresso não altera automaticamente o status.
O mesmo jogo pode possuir experiências diferentes em plataformas diferentes.
O tempo jogado é armazenado em minutos para preservar precisão e facilitar futura integração com a Steam.
A identidade do usuário é obtida através do JWT.
Conclusão

O Milestone 03 foi concluído com a implementação da camada de experiência do jogador no Playlog.

A aplicação deixou de representar apenas a relação entre usuário, jogo e plataforma e passou a registrar também a experiência individual do usuário.

Agora o Playlog consegue representar:

Usuário
   ↓
Biblioteca
   ↓
Jogo + Plataforma
   ├── Status
   ├── Tempo jogado
   ├── Progresso
   └── Avaliação
        ├── Nota
        └── Review


Essa estrutura cria a base para futuras funcionalidades relacionadas ao histórico e à experiência do jogador.

O armazenamento do tempo em minutos também prepara o domínio para uma futura integração com serviços externos, como a Steam, sem necessidade de alterar o conceito de tempo jogado posteriormente.

O sistema passa, portanto, de uma simples biblioteca de jogos para uma plataforma capaz de registrar a experiência do usuário com seus jogos.

Status final: Milestone 03 concluído.