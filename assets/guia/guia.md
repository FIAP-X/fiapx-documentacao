## Guia instrutivo

Acesse a [**collection**](FIAP-X.postman_collection.json) do Postman com todos os endpoints do serviço.

### Passo 1 - cadastro de usuário no Cognito

Para iniciar, é necessário realizar o cadastro de um usuário no AWS Cognito. Utilize o endpoint abaixo para criar um novo usuário.

**POST**  
`https://cognito-idp.us-east-1.amazonaws.com/?Action=AdminCreateUser`

#### Corpo da Requisição

```json
{
  "Username": "deivid@gmail.com",
  "UserPoolId": "",
  "UserAttributes": [
    {
      "Name": "name",
      "Value": "Deivid Cezar"
    }
  ]
}
```

#### Observações

- Após o cadastro, o usuário precisará alterar a senha no primeiro login, conforme o status `FORCE_CHANGE_PASSWORD`.
- As credenciais de autenticação da AWS são necessárias para acessar o endpoint.

### Passo 2 - autenticação do usuário

Após o cadastro, o próximo passo é autenticar o usuário para obter o token de acesso.

**POST**  
`https://cognito-idp.us-east-1.amazonaws.com/?Action=AdminInitiateAuth`

#### Corpo da Requisição

```json
{
  "AuthParameters": {
    "USERNAME": "deivid@gmail.com",
    "PASSWORD": "Senha12#"
  },
  "ClientId": "",
  "UserPoolId": "",
  "AuthFlow": "ADMIN_USER_PASSWORD_AUTH"
}
```

#### Observações

- Certifique-se de usar o `ClientId` e `UserPoolId` corretos conforme configurado no seu ambiente AWS.
- O token de autenticação recebido deverá ser usado nos headers das próximas requisições para acessar os endpoints protegidos.
- A senha deve atender às seguintes exigências de segurança:
    - Mínimo de 8 caracteres;
    - Pelo menos uma letra maiúscula;
    - Pelo menos uma letra minúscula;
    - Pelo menos um número;
    - Pelo menos um símbolo especial.

### Passo 3 - primeira autenticação e alteração de senha

Após o cadastro do usuário, se a senha for exigida a alteração no primeiro login, o usuário precisará enviar um novo password para concluir a autenticação.

**POST**  
`https://cognito-idp.us-east-1.amazonaws.com/?Action=RespondToAuthChallenge`

#### Corpo da Requisição

```json
{
  "ClientId": "",
  "ChallengeName": "NEW_PASSWORD_REQUIRED",
  "ChallengeResponses": {
    "USERNAME" : "deivid@gmail.com",
    "NEW_PASSWORD" : "Senha12#" 
  },
  "Session": ""
}
```

#### Observações

- O valor de `NEW_PASSWORD` deve ser uma senha que atenda às exigências de segurança mencionadas anteriormente.
- A Session é fornecida na resposta anterior do passo 2 de autenticação.
- Após a alteração de senha, o usuário estará autenticado e poderá acessar os endpoints protegidos com o token recebido.

### Passo 4 - gerar URL pré-assinada para upload do vídeo

Com o usuário autenticado, utilize o token de acesso para gerar uma URL pré-assinada que permitirá o upload de um vídeo.

**POST**  
`https://<api_gateway_id>.execute-api.us-east-1.amazonaws.com/prod/generate-url`

#### Authorization

```json
{
  "Authorization": "Bearer <token>"
}
```

#### Observações

- O campo `Authorization` deve conter o Bearer Token obtido no passo de autenticação.

### Passo 5 - upload do vídeo para o S3

Utilize a URL pré-assinada gerada no passo anterior para fazer o upload do vídeo para o S3.

**PUT**
`url pré-assinada do passo anterior`

#### Body

O conteúdo do vídeo em formato binário.

### Passo 6 - consultar status do processamento

Consulte o status do processamento dos vídeos do usuário.

**GET**
`https://<api_gateway_id>.execute-api.us-east-1.amazonaws.com/prod/processamento/{userId}`

#### Authorization

```json
{
  "Authorization": "Bearer <token>"
}
```

### Passo 7 - gerar URL pré-assinada para download do zip com as imagens processadas

O último passo é gerar a URL pré-assinada para fazer o download do resultado do processamento.

**GET**
`https://<api_gateway_id>.execute-api.us-east-1.amazonaws.com/prod/processamento/download?chaveZip={chaveZip}`

#### Authorization

```json
{
  "Authorization": "Bearer <token>"
}
```