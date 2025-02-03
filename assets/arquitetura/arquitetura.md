## Arquitetura

A arquitetura foi projetada para garantir escalabilidade, resiliência e alta disponibilidade na nuvem, 
aproveitando os serviços gerenciados e a infraestrutura como código para facilitar o deployment e a manutenção contínua.

### Desenho da arquitetura

Abaixo, mostra-se a arquitetura dos serviços na AWS.

<p align = "center">
  <img src = arquitetura.svg>
</p>

### Arquitetura hexagonal

Aplicou-se o padrão de arquitetura hexagonal (ports e adapters), que visa isolar o núcleo do domínio das dependências externas, garantindo maior flexibilidade, testabilidade e manutenção do código. 
Nesse contexto, o domínio do processamento foi mantido completamente isolado das camadas de infraestrutura e de interface, permitindo que regras de negócio permaneçam independentes.

### Autenticação

Dado que o sistema deve ser protegido por usuário e senha, implementou-se o AWS Cognito como solução de gerenciamento de identidade e autenticação. 
O Cognito permite criar, autenticar e gerenciar usuários de forma segura, integrando-se facilmente com outros serviços da AWS, como API Gateway e Lambda, para proteger os endpoints da aplicação.

Foram configurados:

- User Pools para o registro e autenticação de usuários, garantindo o controle de acesso baseado em tokens JWT;
- Políticas de senha com requisitos de complexidade (mínimo de caracteres, uso de letras maiúsculas/minúsculas, números e símbolos especiais), reforçando a segurança;
- Fluxo de autenticação com suporte a desafios, como a obrigatoriedade de troca de senha no primeiro login (FORCE_CHANGE_PASSWORD), aumentando a proteção contra acessos não autorizados;
- Tokens de acesso, ID e refresh, permitindo a autenticação segura nas chamadas subsequentes aos serviços protegidos;
- Integração com o API Gateway, utilizando o Cognito Authorizer para validar tokens de forma automática antes de processar as requisições.
- Com essa abordagem, o sistema assegura que apenas usuários autenticados possam acessar os recursos sensíveis, mantendo a conformidade com boas práticas de segurança.

### Upload com url pré-assinada

O sistema requer a funcionalidade de upload de arquivos de vídeo pelos usuários, com segurança, escalabilidade e eficiência. 
Considerando o volume potencial de uploads e a necessidade de proteger o armazenamento no Amazon S3, foi adotada uma abordagem que minimiza a sobrecarga no backend e garante controle de acesso adequado.

Optou-se pelo uso de URLs pré-assinadas (pre-signed URLs) para o upload direto dos arquivos no Amazon S3. A arquitetura implementada segue o fluxo abaixo:

- API Gateway: recebe a solicitação do cliente para gerar a URL pré-assinada. O endpoint é protegido via AWS Cognito para garantir que apenas usuários autenticados possam gerar URLs.
- AWS Lambda: função Lambda é acionada pelo API Gateway para gerar a URL pré-assinada. Essa função utiliza as permissões adequadas para interagir com o S3 e definir o tempo de expiração da URL.
- Amazon S3: com a URL pré-assinada, o cliente realiza o upload diretamente no bucket S3, sem passar pelo servidor backend. Isso reduz a latência, melhora o desempenho e minimiza o custo de transferência de dados.

Além disso, foi configurado um lifecycle policy no Amazon S3 para gerenciar automaticamente a expiração e a transição de objetos, garantindo que os arquivos enviados sejam arquivados ou removidos conforme políticas de retenção específicas, otimizando o uso de armazenamento e reduzindo custos a longo prazo.

### Processamento dos vídeos em imagens

O sistema precisa processar vídeos carregados pelos usuários, convertendo-os em imagens e salvando os resultados em arquivos ZIP. 
Para isso, foi adotada uma arquitetura baseada em AWS SQS e ECS Fargate, com armazenamento em Amazon S3 e persistência de status no RDS. 
Essa solução visa atender a diversos requisitos de escalabilidade, disponibilidade e persistência de dados, assegurando o processamento eficiente e resiliente, mesmo em cenários de alta demanda.

- AWS SQS: a fila SQS recebe mensagens com informações sobre os vídeos que precisam ser processados. Essa solução permite desacoplar a ingestão dos vídeos e o processamento, garantindo que o sistema possa escalar de forma eficiente e segura, sem perda de requisições durante picos de tráfego.
- ECS Fargate: o processamento dos vídeos é realizado por containers ECS Fargate, que permitem uma execução sem servidor, escalando automaticamente conforme a demanda. Com o ECS Fargate, cada vídeo é processado em paralelo, permitindo o processamento simultâneo de múltiplos vídeos. O autoscaling do ECS é configurado para garantir que novos containers sejam provisionados conforme a carga aumenta, evitando gargalos e otimizando o uso de recursos.
- Armazenamento no S3: os arquivos ZIP com as imagens extraídas dos vídeos são armazenados em um bucket S3 dedicado, permitindo fácil acesso e download pelo usuário. A arquitetura do S3 oferece alta durabilidade e disponibilidade dos arquivos.
- Persistência de Status no RDS: o status do processamento de cada vídeo é armazenado em um banco de dados RDS, garantindo que o progresso do processamento seja monitorado e acessível. Esse status é atualizado ao longo do fluxo de processamento, permitindo que o sistema forneça uma listagem de status dos vídeos para os usuários.

### Gerenciamento de falhas com notificação via e-mail

Em caso de falha no processamento de um vídeo, foi implementada uma Dead Letter Queue (DLQ) para garantir que as mensagens que não puderam ser processadas sejam armazenadas de forma segura e possam ser analisadas posteriormente. Quando uma falha é detectada, a mensagem é movida para a DLQ, onde pode ser tratada adequadamente.

Para garantir que o usuário seja informado sobre a falha, uma Lambda é configurada para consumir as mensagens da DLQ. Esta função Lambda é responsável por enviar uma notificação por e-mail ao usuário, utilizando o Amazon SES (Simple Email Service), informando-o sobre o erro ocorrido no processamento de seu vídeo.

### Download do resultado do processamento

Os vídeos processados são acessados pelos usuários através da geração de URLs pré-assinadas. Essa abordagem garante que apenas usuários autenticados e autorizados possam acessar seus respectivos vídeos, mantendo o controle sobre quem pode visualizar os conteúdos. A URL pré-assinada tem um tempo de expiração limitado, o que aumenta a segurança, impedindo acessos não autorizados após um período definido.

Em conformidade com a Lei Geral de Proteção de Dados (LGPD), que estabelece diretrizes rigorosas sobre o tratamento de dados pessoais, incluindo a necessidade de garantir a segurança e a privacidade dos dados dos usuários, a arquitetura foi projetada para proteger a privacidade e a confidencialidade dos vídeos.

Para garantir que os vídeos e seus dados não sejam acessados indevidamente, todos os buckets S3 são configurados como privados, com acesso restrito a funções específicas e controladas por permissões rigorosas. O acesso ao conteúdo do vídeo só é possível através da geração da URL pré-assinada, garantindo que a privacidade do usuário seja preservada, mesmo durante a transmissão ou o armazenamento.