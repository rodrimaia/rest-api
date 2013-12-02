# Índice

- [Introdução](#introducao)
- [Funcionamento geral](#funcionamento-geral)
- [Autenticação](#autenticacao)
- [Upload de documentos](#upload-de-documentos)
- [Download de um documento](#download-de-um-documento)
- [Criação de usuários corporativos](#criacao-de-usuarios-corporativos)
- [Hooks](#hooks)

# <a name="introducao"></a>Introdução

A Clicksign é uma solução online para enviar, guardar e assinar documentos, com validade jurídica. Foi criada para facilitar, reduzir custo e aumentar a segurança e compliance do processo de assinatura e _workflow_ de documentos. 

A Clicksign pode ser acessada em https://desk.clicksign.com. 

O propósito desta **REST API** é prover meios para que nossos clientes adequem a Clicksign aos seus processos e sistemas p. ex. automatizar tarefas, desenhar fluxos de assinatura, e definir _workflow_. 

Qualquer linguagem de programação compativel com requisições **HTTP / JSON** cumpre os requisitos necessários para consumir os serviços desta API. Assim, com pouco esforço de programação é possível integrar desde scripts shell até sistemas de ERP.

# <a name="funcionamento-geral"></a>Funcionamento geral

Uma _REST API_ é composta, basicamente, por dois elementos: um **cliente** e um **servidor**. O cliente sempre inicia a comunicação mediante requisição HTTP. O servidor sempre finaliza a comunicação respondendo à requisição.

<p align="center">
  <img src="https://raw.github.com/clicksign/rest-api-v2/master/images/client-server.png" />
</p>

As mensagens HTTP são compostas por uma linha inicial, um conjunto de cabeçalhos e um corpo. A linha inicial difere nas requisições e nas respostas, o cabeçalho compartilha parâmetros em comum e parâmetros específicos, e o corpo é completamente dependente de cada mensagem, podendo até ser nulo.

A requisição, em sua linha inicial, indica o **método**, o **caminho**, e a **versão do protocolo**. O método e o caminho são essenciais em uma _REST API_ uma vez que ambos indicam a ação a ser executada no servidor.

A resposta, em sua linha inicial, indica a **versão do protocolo**, o **status**, e contém uma **mensagem informativa**. O código de status é essencial para o cliente saber se a ação foi devidamente executada no servidor.

A documentação de cada função da API determina o método e o caminho a ser utilizado, e o significado do corpo e de cada status da resposta.

**Atenção:** Toda a comunição cliente/servidor é feita através de HTTP sobre SSL/TLS (HTTPS). Requisições em HTTP simples resultam em redirecionamentos (304) para o protocolo HTTPS.

Todas as requisições da _REST API_ são feitas para `api.clicksign.com`.

## Exemplo de requisição

```http
GET /documents HTTPS/1.1
Host: api.clicksign.com
Accept: application/json
```

- Método: GET
- Caminho: /documents
- Versão: 1.1
- Cabeçalhos: Host, Accept
- Corpo: vazio

## Exemplo de resposta

```http
HTTP/1.1 200 OK
Content-Type:application/json
Connection: Keep-Alive
```

```json
[
  {
    "id": "abcd..."
  },

  {
    "id": "..."
  }
]
```

- Versão: 1.1
- Status: 200
- Mensagem: OK
- Cabeçalhos: Content-Type, Connection
- Corpo: [{...

# <a name="autenticacao"></a>Autenticação

A Clicksign utiliza duplo fator de autenticação para aumentar a segurança de suas transações. Autenticações que utilizam duplo fator geralmente são baseadas em algo que o cliente _conhece_ e algo que o cliente _possui_. No caso da API os fatores são:

1. Conhecer um par **identificação** e **segredo**
1. Possuir um endereço **IP** específico

O primeior fator da autenticação é feito através de 2 parâmetros: **api_id** e **api_secret**. O parâmetro `api_id` define qual cliente está fazendo a requisição. O parâmetro `api_secret` define o senha que será utilizada na verificação de acesso à API. Ambos os parâmetros devem ser enviados no **caminho** da requisição. Portanto, toda requisição deverá conter no _path_ `?api_id=string-da-key&api_secret=string-do-secret`.

**Atenção:** Os parâmetros de autenticação devem ser enviados a cada requisição feita pelo cliente. Como esses parâmetros são comuns a todos as funções da API, eles serão omitidos das documentações.

O segundo fator da autenticação é realizado automaticamente pelo servidor da Clicksign, que verifica se o **IP** de origem da requisição está dentro de uma lista de endereços previamente cadastrados para determinado cliente.


# <a name="upload-de-documentos"></a>Upload de documentos

O processo de envio de um documento para o servidor contempla a criação de um arquivo de **log**c ontendo informações de _upload_, usuário, etc, anexado a uma cópia do documento "carimbada" com um **número de série**. Ao final do processo haverá 2 arquivos nos servidores da Clicksign: documento original e arquivo de log anexado a uma cópia carimbada do documento. Enquanto o log e a cópia carimbada são gerados a requisição *não fica bloqueada*. O _status_ do documento será _working_ enquanto o processo ocorre. Após concluído, o _status_ será _open_.

* **Method:** POST
* **Path:** /documents
  - **Content-Type:** multipart/mixed; boundary=frontier
  - **Accept**: application/json
* **Corpo:**
  - **Content-Type:** application/octet-stream
  - **Content-Transfer-Encoding:** base64

  ```
  --frontier--
  PGh0bWw+CiAgPGhlYWQ+CiAgPC9oZWFkPgogIDxib2R5PgogICAgPHA+VGhpcyBpcyB0aGUg
  Ym9keSBvZiB0aGUgbWVzc2FnZS48L3A+CiAgPC9ib2R5Pgo8L2h0bWw+Cg==
  --frontier--
  ```

## Exemplo de resposta

```http
HTTP/1.1 200 OK
Content-Type:application/json
Connection: Keep-Alive
```

```json
{
  "key": "0123-4567-89ab-cdef",
  "status": "working"
}
```

## Resposta 4XX

Caso o cliente utilize parâmetros inválidos, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

```json
{ "message": "Invalid parameters." }
```

## Resposta 5XX

Caso ocorra qualquer tipo de falha no servidor, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

```json
{ "message": "Server error." }
```


# Criação de lista de assinatura

É possível criar uma lista de assinatura e enviá-la a outras pessoas em uma única ação. Para isso, é necessário que estejam presentes os campos que especificam o documento, os signatários, e a mensagem.

* **Method:** POST
* **Path:** /documents/:key/signature_list
* **Cabeçalhos:**
  - **Content-Type:** application/json
  - **Accept**: application/json
* **Corpo:**

  ```json
  {
    "signers": [
      { "email": "foo@example.com", "action": "sign" },
      { "email": "bar@example.com", "action": "sign_as_witness" }
    ],

    "message": {
      "recipients": [ "foo@example.com", "bar@example.com" ],
      "body": "Hi guys, please sign this document."
    }
  }
  ```

## Especificando signatários

Para criar uma lista de assinatura, adicionar signatários ao documento e iniciar o processo de assinatura automaticamente, deve-se adicionar um campo `signers` ao JSON. Caso não haja o campo `signers` ou ele seja `null`, o documento não possuirá lista de assinatura definida.

O campo `signers` deverá ser um `Array` contendo os signatários. Cada signatário é especificado através de e-mail e ação, sendo os respectivos campos `email` e `action`.

Os possíveis campos de `action` são:
- sign
- approve
- sign_as_party
- sign_as_witness
- sign_as_intervenient

```json
{
  "signers": [
    { "email": "foo@example.com", "action": "sign" },
    { "email": "bar@example.com", "action": "sign_as_witness" }
  ]
}
```

## Especificando a mensagem

Para especificar a mensagem a ser enviada são necessários dois campos: `recipients` e `body`.

O campo `recipients` é um `Array` obrigatório com tamanho mínimo de `1`. Nenhuma mensagem será enviada caso o campo `recipients` não exista, seja `null` ou tenha tamanho igual a `0`.

O campo `body` especifica o corpo da mensagem, é opcional e caso presente deve ser do tipo `String`.

```json
{
  "message": {
    "recipients": [ "foo@example.com", "bar@example.com" ],
    "body": "Hi guys, please sign this document."
  }
}
```


# <a name="download-de-um-documento"></a>Download de um documento

Retorna um arquivo _ZIP_ contendo os 2 arquivos resultantes do processamento: arquivo original, log concatenado a uma cópia carimbada do arquivo.

* **Method:** GET
* **Path:** /documents/:key
* **Cabeçalhos:**
  - **Accept**: application/zip
* **Corpo:** _vazio_

## Resposta 200

Caso não ocorra nenhuma falha na requisição, o corpo da resposta será o arquivo _ZIP_.

* **Cabeçalhos**:
  - **Content-Type:** application/zip
* **Corpo:**

  ```
  PGh0bWw+CiAgPGhlYWQ+CiAgPC9oZWFkPgogIDxib2R5PgogICAgPHA+VGhpcyBpcyB0aGUg
  ...
  Ym9keSBvZiB0aGUgbWVzc2FnZS48L3A+CiAgPC9ib2R5Pgo8L2h0bWw+Cg==
  ```

## Resposta 4XX

Caso o cliente utilize parâmetros inválidos, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

  ```json
  {
    "message": "Parâmetros inválidos."
  }
  ```

## Resposta 5XX

Caso ocorra qualquer tipo de falha no servidor, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

  ```json
  {
    "message": "Server error."
  }
  ```


# <a name="criacao-de-usuarios-corporativos"></a>Criação de usuários corporativos

* **Method:** POST
* **Path:** /registration
* **Cabeçalhos:**
  - **Content-Type:** application/json
  - **Accept**: application/json
* **Corpo:**
  ```json
  {
    "person": {
      "name": {
        "given_name": "John",
        "additional_name": "August",
        "family_name": "Doe",
        "honorific_suffix": "III"
      },

      "documentation": {
        "country": "br",
        "kind": "cpf",
        "value": "999.999.999-99"
      },

      "phone": {
        "country": "br",
        "number": "99-9-9999-9999"
      }
    }
  }
  ```

Para especificar a criação de um usuário corporativo a requisição json deve seguir o formato especificado acima.

<table>
  <thead>
    <tr>
      <th>Campo</th>
      <th>Tipo</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>person.name.given_name</td>
      <td>String com até 50 caracteres</td>
    </tr>

    <tr>
      <td>person.name.additional_name</td>
      <td>String com até 50 caracteres</td>
    </tr>

    <tr>
      <td>person.name.family_name</td>
      <td>String com até 50 caracteres. A quantidade total de sobrenomes (additional_name + family_name + honorific_suffix, separados por espaço) não deve ultrapassar a quantidade de <b>6 nomes</b></td>
    </tr>

    <tr>
      <td>person.name.honorific_suffix</td>
      <td>String com até 50 caracteres</td>
    </tr>

    <tr>
      <td>person.documentation.country</td>
      <td>"br"</td>
    </tr>

    <tr>
      <td>person.documentation.kind</td>
      <td>"cpf"</td>
    </tr>

    <tr>
      <td>person.documentation.value</td>
      <td>String com 11 digítos com ou sem pontuação, exemplos válidos: "99999999999", "999.999.999-99"</td>
    </tr>

    <tr>
      <td>person.phone.country</td>
      <td>"br"</td>
    </tr>

    <tr>
      <td>person.phone.number</td>
      <td>String com 10 ou 11 digítos com ou sem pontuação, onde os dois primeiros representam o **ddd** e os últimos 8 ou 9 digítos representam ou número celular</td, exemplos válidos: "99-9999-9999", "99-9-9999-9999", "99999999999", "9999999999"</td>
    </tr>
  </tbody>
</table>


## Resposta 200

Caso não ocorra nenhuma falha na requisição, o corpo da resposta será um _JSON_ contendo as informações do cadastro de um usuário corporativo.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

  ```json
  {
    "registration": {
      "url": "https://desk.clicksign.com/registration?uuid=...."
    }
  }
  ```

## Resposta 4XX

Caso o cliente utilize parâmetros inválidos, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

  ```json
  {
    "message": "Parâmetros inválidos."
  }
  ```

## Resposta 5XX

Caso ocorra qualquer tipo de falha no servidor, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

  ```json
  {
    "message": "Server error."
  }
  ```


# <a name="hooks"></a>Hooks

É possível que a Clicksign notifique outras aplições à respeito de determinados eventos, p. ex., documento completamente assinado.

Para isso, a Clicksign dispõe de um sistema de **hooks** que realizam chamadas HTTP para outras aplicações. As _hooks_ são definidas por usuário, portanto cada usuário deve configurar as _hooks_ com os parâmetros que deseja.

Os parâmetros possíveis são:

- **URL**: caminho completo, incluíndo protocolo
- **Método**: GET, POST, PUT, DELETE, PATCH
- **Content-Type**: application/json

É anexado ao corpo da requisição uma representação em _JSON_ do evento que a disparou, p. ex., em evento de documento completamente assinado é anexado um _JSON_ do documento, da lista de assinatura e das assinaturas do documento.

Os tipos de _hooks_ implementados até o momento são:

- documento pendente de assinatura
- documento completamente assinado

## Configuração de hooks

Cadastra um _hook_ para um determinado usuário.

* **Method:** POST
* **Path:** /users/:id/hooks
* **Corpo:**
  ```json
  {
    "pending": {
      "url": "https://example.com/signed/123",
      "method": "POST"
    },

    "completed": {
      "url": "https://example.com/completed/123",
      "method": "POST"
    }
  }
  ```

## Resposta 200

Caso não ocorra nenhuma falha na requisição, a resposta será apenas o status 200 e seu corpo será vazio.

## Resposta 4XX

Caso o cliente utilize parâmetros inválidos, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

  ```json
  {
    "message": "Parâmetros inválidos."
  }
  ```

## Resposta 5XX

Caso ocorra qualquer tipo de falha no servidor, o corpo da resposta será um _JSON_ contendo uma mensagem de erro.

* **Cabeçalhos**:
  - **Content-Type:** application/json
* **Corpo:**

  ```json
  {
    "message": "Server error."
  }
  ```
