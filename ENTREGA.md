# Tech Challenge - Fase 1: Plataforma "ToggleMaster"


## Participantes

- Bruno Silva Baena de Souza - baena@bb.com.br

## Link Documentação
- [Diagrama Infraestrutura AWS](https://www.figma.com/design/2hyW0p7oEpU8YikRapTQaE/Tech-Challenge---Fase-1---Diagrama-AWS?m=auto&t=6IOtCGMOO9BQK5ZP-1)
- [Video Demonstração](https://youtu.be/HXe9FFIKNWw)

## Análise da Aplicação

- O arquivo `app.py` contém o código fonte principal de uma API desenvoldida em python usando Flask, ele é considerado uma aplicação monitícia porque contém em um único projeto toda a interface, lógica de negócio, tratamento de dados, e persistência estão fortemente acoplados em um único projeto (nesse caso, um arquivo), não sendo possível alterar uma camada (de persistência, por exemplo), sem ter que mudar a estrutura inteira do arquivo.
  Sendo uma aplicação inicial, com um escopo reduzido, o monolito se aplica pois é fácil de desenvolver, e pode-se rapidamente disponibilizar um MVP para o cliente final, porém, a medida que mais funcionalidades são incluídas, esse paradigma se torna dificil de manter e escalar.

- O arquivo `Dockerfile` é responsável pela criação da imagem docker do projeto para ser executado, no qual é utilizada uma imagem base do python (`python:3.9-slim`), a instalação das dependencias do projeto (`requirements.txt`), e a definição de execução da aplicação na porta 5000. Esta imagem também possui um cliente postgresql que verifica quando o banco de dados está pronto para ser utilizado (pelo script `entrypoint.sh`)

- O arquivo `docker-compose.yml` é o responsável por levantar os containers dos serviços, tanto o serviço da API, rodando na porta 5000, quanto o serviço de banco de dados, rodando na porta 5432. Neste arquivo também foram definidas as varíáveis de ambiente contendo também os parâmetros de conexão do banco de dados.

## Metodologia 12-Factor App

Breve análise da aplicação `toogle-master-monolith` em relação a cada um dos 12 fatores

### I. Base de Código
O código fonte da aplicação é gerenciado por um sistema de controle de versão git, em uma única base de código, com uma única aplicação sendo compartilhada.

### II. Dependências
A aplicação possui um arquivo específico de declaração de dependencias python, através do arquivo `requirements.txt`, além de gerar uma imagem docker para execução, o que garante que todas as dependências e ferramentas necessárias para a execução da aplicação estão disponíveis e configuradas para a distribuição em qualquer plataforma e ambiente

### III. Configurações
A aplicação faz uso de variáveis de ambiente para a conexão com bando de dados, definindo nome de variáveis específicas e não atreladas ao ambiente de execução

### IV. Serviços de Apoio
Quanto aos recursos anexados, a aplicação possui um recurso de banco de dados, sem qualquer acoplamento com um fornecedor específico, sendo possível trocasr de recurso apenas mudando a configuração de conexão

### V. Construa, lance, execute
A aplicação possui os estágios de construção e execução bem definidos, durante a construção da imagem docker as dependências são obtidas e todo o sistema é transformado em uma imagem que pode ser versionada e executada a qualquer momento, sem possibilidade de alteração do conteúdo da imagem em execução

### VI. Processos
A natureza da aplicação é stateless. Ela recebe requisições, processa, armazena em recurso externo (banco de dados) e finaliza a execução. Sucessivas requisições não dependem do estado de outras requisições.

### VII. Vínculo de porta
A aplicação usa o servidor Gunicorn embutido e é disponibilizada na porta 5000, ainda que esteja em conformidade com este principio, o ideal seria utilizar uma variável de ambiente para que o binding pudesse ser especificado em tempo de execução

### VIII. Concorrência
A aplicação por ser stateless, possui excelente possibilidade de escalar horizontalmente, pois uma requisição não depende das outras. Da mesma forma, na execução da aplicação, ao usar o gunicorn, estamos definindo um tipo específico de processo web, porém da forma como está configurado o Gunicorn utiliza apenas 1 worker, o que limita a execução a apenas 1 requisição por vez, sendo um ponto de melhoria

### IX. Descartabilidade
Aqui temos alguns alguns pontos de melhoria. A inicialização da aplicação está vinculada à disponibilidade do banco de dados, o que não está de acordo com o princípio de inicialização rápida. Do ponto de vista do desligamento, o Gunicorn poderia ser configurado para não desligar abruptamente e aguardar um tempo para que as requisições em execução tivessem a chance de terminar o processamento

### X. Dev/prod semelhantes
O uso de containers promove a diminuição da diferença entre os ambientes de desenvolvimento e produção, além de facilitar deploys contínuos

### XI. Logs
A aplicação trata o log como um fluxo de eventos, ao enviar as mensagem para o fluxo de saída com o uso da função `printf`

### XII. Processos de Admin
Apesar de definir um task administrativa para a inicialização do banco de dados, a execução dessa tarefa não é pontual, pois está acoplada à inicialização do container

## Estimativa Custo AWS
![Estimativa Custo](https://github.com/brunobaena/toggle-master-monolith/blob/42bcd5223c0b1c831e31ced3c2cb8074a100c558/estimativa_custo_aws.png)
![Estimativa Custo Detalhado](https://github.com/brunobaena/toggle-master-monolith/blob/42bcd5223c0b1c831e31ced3c2cb8074a100c558/estimativa_custo_detalhado_aws.png)


## Resumo Desafios

- O primeiro desafio encontrado foi o de fazer a aplicação rodar localmente, para isso foi necessário instalar o docker localmente, juntamente com o plugin docker compose, fora isso, o projeto já está preparado para rodar locamente, sem maiores dificuldades.
  Depois de iniciar o projeto com `docker compose up --build`, é feita a construção da image a partir do Dockerfile, e o pull das imagens base (python e postgres), a partir disso a aplicação já está rodando e é possível executar os comandos `curl` para verificação do funcionamento da aplicação

- Após isso, o desafio é implantar este projeto na núvem da AWS, onde é necessário provisionar e configurar toda a infraestrutura necessária para a execução do projeto, conforme os passos a seguir:
  1. É criada uma VPC para a aplicação com o IP base (`10.0.0.0/16`) juntamente com a criação de duas subnets, sendo uma pública e uma privada, nos endereços `10.0.1.0/24` e `10.0.2.0/24`, respectivamente.
  2. São criados dois Security Groups, para a camada de segurança das subnets. Um Security Group para a subrede pública que aceita conexões da porta 5000 (aplicação) e 22 (ssh). O outro Security Group permite a conexão para a subnet privada, apenas para conxões feitas de dentro da subrede pública
  3. É criado um Internet Gateway para conectar a subrede pública com a internet.
  4. É criada uma instância do EC2 que ficará responsável pela aplicação que será executada.
  5. É criada uma instância do RDS para armazenamento dos dados, juntamente com as configurações de usuário e senha para conexão. O usuário e senha são armazenados em um AWS Secret Manager
  6. É criada uma imagem docker da aplicação de forma a simplificar o deploy da mesma, sem a necessidade de instalar aplicativos e pacotes de dependencias a cada EC2 criado.
  7. É Criado um cluster ECS, self-managed, usando o EC2 criado anteriormente para gerenciar a execução da imagem docker da aplicação.

- Uma das grandes dificuldades encontradas para rodar a aplicação usando essa arquitetura proposta foi entender como fazer o ECS rodar um container usando a instância EC2 previamente criada, depois de muita busca, entendeu-se a necessidade de configurar a instância EC2 para se comunicar com o ECS através de uma agente proprio para esta integração. A partir desse entendimento, subir a aplicação fica muito mais fácil, pois a imagem já contém tudo que é necessário para rodar a aplicação.
