# Especificação - Document Management System

## 1. Objetivo

Entregar uma aplicação web simples para enviar, listar e baixar documentos armazenados no filesystem local, com metadados mantidos em memória.

## 2. Escopo

### Dentro do escopo

- Enviar um documento por requisição.
- Listar os documentos registrados durante a execução atual do backend.
- Baixar um documento pelo identificador.
- Registrar um identificador demonstrativo de dono nos metadados.
- Interface React para upload, listagem e download, consumindo a API pelo proxy `/api` do Vite.

### Fora do escopo

- Armazenamento externo ou em nuvem.
- Banco de dados ou persistência durável dos metadados.
- Autenticação, autorização ou isolamento de documentos entre usuários.
- Versionamento, edição, exclusão, busca, paginação ou compartilhamento de documentos.
- Restrição por extensão ou tipo MIME.

## 3. Requisitos funcionais

| ID | Requisito | Critério de aceite |
| --- | --- | --- |
| RF-01 | O sistema deve aceitar o upload de um documento por vez. | Uma requisição `multipart/form-data` com o campo `file` e arquivo de até 10 MB retorna `201` e os metadados criados. |
| RF-02 | O sistema deve rejeitar upload sem arquivo ou acima do limite. | Ausência do campo retorna `400`; arquivo maior que 10 MB retorna `413`; nenhuma entrada é adicionada à listagem. |
| RF-03 | O sistema deve registrar metadados para cada upload aceito. | O registro contém `id`, `originalName`, `size`, `uploadedAt` e `owner`, com ID único, tamanho em bytes e data/hora ISO 8601. |
| RF-04 | O usuário deve poder listar os documentos registrados. | `GET /documents` retorna `200` e um array JSON, incluindo array vazio quando não houver documentos registrados. A lista é ordenada do mais recente para o mais antigo. |
| RF-05 | O usuário deve poder baixar um documento pelo identificador. | Um ID registrado com arquivo disponível retorna o conteúdo como anexo; ID não registrado ou arquivo ausente retorna `404`. |
| RF-06 | O sistema deve manter arquivos e metadados separados por responsabilidade. | O arquivo é salvo em `backend/storage`; o repositório mantém metadados em memória e o nome físico interno não é exposto na API. |

## 4. Requisitos não funcionais

| ID | Requisito | Critério de aceite |
| --- | --- | --- |
| RNF-01 | Os arquivos devem ser armazenados exclusivamente no filesystem local com `multer` e `diskStorage`. | Todo arquivo aceito fica sob `backend/storage`; não há integração com serviços externos. |
| RNF-02 | Os metadados devem permanecer em memória nesta fase. | Ao reiniciar o backend, a lista volta vazia; arquivos já gravados podem permanecer sem metadados associados. |
| RNF-03 | Configurações operacionais devem usar variáveis de ambiente quando aplicável. | A porta do backend é configurável por `PORT`; o diretório de armazenamento permanece fixo em `backend/storage` para atender à restrição local. |
| RNF-04 | A arquitetura do backend deve separar transporte, regras de negócio e persistência. | O fluxo de dependências é `routes -> controllers -> services -> repositories`; camadas internas não dependem de Express. |
| RNF-05 | O backend deve ser implementado em JavaScript CommonJS e testado com `node:test`. | A suíte do backend é executável por `npm test` dentro de `backend`. |
| RNF-06 | A interface deve seguir a stack e os padrões definidos para o frontend. | React com componentes funcionais e hooks; chamadas HTTP feitas por `fetch` usando o prefixo `/api`. |
| RNF-07 | O limite por arquivo deve ser de 10 MB. | Arquivos acima de 10 MB são rejeitados antes de criar metadados. O limite é por arquivo, não por soma de requisições. |
| RNF-08 | O nome enviado pelo usuário não deve determinar o caminho físico. | O backend gera um nome de armazenamento único; o nome original serve apenas como metadado e nome sugerido no download. |
| RNF-09 | Os arquivos devem ser entregues como download, não renderizados como conteúdo inline. | A resposta de download define `Content-Disposition: attachment` e não expõe caminhos locais. |

## 5. Modelo de dados (metadados do documento)

### Registro público

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | string | Identificador único, gerado pelo backend. UUID é o formato recomendado. |
| `originalName` | string | Nome original fornecido no upload, mantido para exibição e sugestão de nome no download. |
| `size` | number | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | Data e hora do upload em ISO 8601, preferencialmente UTC (`toISOString()`). |
| `owner` | string | Identificador demonstrativo fixo `local-user`; não representa uma identidade autenticada. |

### Referência interna de armazenamento

O repositório precisa associar cada registro ao nome físico gerado para o arquivo. Essa referência é interna, não faz parte dos metadados públicos e não deve conter um caminho fornecido pelo cliente. O caminho absoluto é resolvido pelo backend dentro de `backend/storage`.

Os metadados existem apenas na memória do processo. O reinício do backend elimina a associação entre IDs e arquivos; arquivos no diretório local não são removidos automaticamente nesta versão.

## 6. Contratos de API

As rotas abaixo são caminhos do backend. No frontend, o cliente usa `/api` como prefixo; o proxy do Vite remove esse prefixo antes de encaminhar a chamada ao backend. Portanto, o frontend chama, por exemplo, `/api/documents`, enquanto o backend recebe `/documents`.

### Formato comum de erro

Erros conhecidos retornam JSON com a mesma estrutura:

```json
{
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "Documento não encontrado."
  }
}
```

`code` é estável para tratamento pelo cliente; `message` é uma mensagem legível em português. Respostas de erro não devem incluir stack trace, caminhos locais ou detalhes internos do filesystem.

### `POST /upload`

- **Objetivo:** armazenar um documento e registrar seus metadados.
- **Entrada:** `multipart/form-data`, um único campo de arquivo chamado `file`.
- **Validação:** o arquivo é obrigatório e deve ter até 10 MB. Qualquer tipo MIME e extensão são aceitos nesta fase.
- **Sucesso:** `201 Created`, `Content-Type: application/json`.

```json
{
  "id": "b7f4af10-8f72-4d53-a1ea-3a4da66345df",
  "originalName": "relatorio.pdf",
  "size": 24576,
  "uploadedAt": "2026-10-28T14:30:00.000Z",
  "owner": "local-user"
}
```

| Status | Código de erro | Quando ocorre |
| --- | --- | --- |
| `400` | `FILE_REQUIRED` | Campo `file` ausente ou requisição multipart inválida. |
| `413` | `FILE_TOO_LARGE` | Arquivo acima de 10 MB. |
| `500` | `STORAGE_ERROR` | Falha ao gravar o arquivo ou registrar os metadados. |

Se uma falha ocorrer durante a gravação, o backend não deve criar um registro de metadados. O nome físico é gerado pelo backend e não deriva diretamente do nome enviado.

### `GET /documents`

- **Objetivo:** listar os metadados registrados na execução atual.
- **Entrada:** nenhuma.
- **Sucesso:** `200 OK`, `Content-Type: application/json`, corpo como array de registros públicos. Ordenação por `uploadedAt` decrescente; em empate, por `id` crescente.
- **Lista vazia:** retorna `200` com `[]`.
- **Erros:** `500` com código `DOCUMENT_LIST_ERROR` se a listagem falhar.

Não há filtro por dono: `owner` é apenas demonstrativo e não fornece isolamento ou privacidade entre usuários.

### `GET /documents/:id/download`

- **Objetivo:** baixar o arquivo associado a um ID registrado.
- **Entrada:** `id` no caminho.
- **Sucesso:** `200 OK`, corpo binário, `Content-Type: application/octet-stream` e `Content-Disposition: attachment` com o nome original devidamente codificado. O nome original não deve ser concatenado sem tratamento em cabeçalhos HTTP.
- **Erros:** `404` com código `DOCUMENT_NOT_FOUND` se o ID não estiver registrado ou se o arquivo associado não existir; `500` com código `STORAGE_ERROR` para outras falhas de leitura.

O caminho do arquivo é resolvido exclusivamente a partir do registro interno e do diretório local configurado no código. Um ID arbitrário não pode ser usado como caminho de filesystem.

## 7. Decisões arquiteturais

### Backend

- **Routes:** declaram `POST /upload`, `GET /documents` e `GET /documents/:id/download`, configuram o middleware de upload e delegam aos controllers.
- **Controllers:** adaptam requisição e resposta HTTP, validam dados básicos de entrada e traduzem erros conhecidos para status e JSON definidos nesta especificação.
- **Services:** implementam as regras de negócio: limite e aceitação do upload, geração de ID/data/dono, registro, listagem e obtenção de arquivo para download.
- **Repositories:** encapsulam a coleção em memória de metadados e a associação ao nome físico do arquivo.
- **Armazenamento:** `multer` com `diskStorage` grava em `backend/storage`. O middleware aceita um arquivo por requisição, limita o tamanho a 10 MB e usa nome físico gerado com segurança.

### Frontend

- React + Vite, com componentes funcionais e hooks.
- Um serviço de API centraliza chamadas `fetch` para `/api/upload`, `/api/documents` e `/api/documents/:id/download`.
- A interface permite selecionar e enviar um arquivo, apresentar estado de carregamento/erro, atualizar a lista e iniciar o download.

### Decisões e limitações

- `owner` tem valor fixo `local-user`; não há autenticação, autorização nem isolamento entre usuários.
- Os arquivos permanecem no disco entre reinícios, mas os metadados não. Um arquivo sem metadados não pode ser listado nem baixado pela API.
- Não há recuperação automática de metadados, limpeza de arquivos órfãos ou transação entre escrita do arquivo e memória.
- São aceitos quaisquer tipos de arquivo, mas downloads são sempre enviados como anexo. A aplicação não valida conteúdo nem examina arquivos.
- O prefixo `/api` existe no cliente frontend/proxy; os caminhos dos contratos backend não incluem esse prefixo.

## 8. Plano de execução

As etapas abaixo descrevem trabalho futuro. Esta especificação não implementa os arquivos de backend ou frontend.

1. **Backend - estrutura e contratos:** criar rotas, controllers, services e repositories em `backend/src`; definir erros comuns e testes unitários do repositório em memória.
2. **Backend - upload:** configurar `multer` com `diskStorage` em `backend/storage`, nome físico gerado, campo `file`, limite de 10 MB e testes para sucesso, arquivo ausente, excesso de tamanho e falha de armazenamento.
3. **Backend - listagem e download:** implementar ordenação estável, recuperação pelo ID e entrega como anexo; testar lista vazia, múltiplos registros, ID inexistente e arquivo local ausente.
4. **Backend - integração:** registrar o router em `backend/src/app.js`, cobrir contratos HTTP e erros com `node:test` e executar `npm test` em `backend`.
5. **Frontend:** implementar serviço `fetch`, componentes de upload/listagem/download e estados de carregamento e erro; consumir a API via prefixo `/api` já encaminhado pelo Vite.
6. **Validação integrada:** iniciar backend e frontend, verificar upload, listagem e download pela interface, validar o limite de tamanho e confirmar o comportamento após reinício do backend.

## Premissas para esta versão

- A porta do backend é controlada por `PORT`; o diretório de armazenamento é fixo em `backend/storage`.
- O limite de upload é 10 MB por arquivo, com qualquer tipo permitido.
- O usuário é demonstrativo e compartilhado; autenticação e privacidade por usuário exigem uma especificação futura.
- Metadados em memória e possíveis arquivos órfãos após reinício são limitações aceitas do MVP.