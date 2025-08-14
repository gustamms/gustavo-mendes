---
title: "Usando Localstack com S3"
draft: false
date: 2025-08-25T09:16:45.000Z
description: "Como e porque você deveria usar essa maravilha chamada Localstack."
categories:
  - AWS
---

Bom, aqui é meu espaco onde coloco algumas coisas que com toda certeza terei que voltar posteriormente para rever, pois não vou armazenar isso na minha cabeca.

Mas atualmente tivemos um problema nas credenciais da AWS no geral e tivemos que migrar tudo para IAM Roles, não mais usar chaves, com isso, achei que fosse somente adicionar na Role do servico, MAS POREM, achei isso -> [Link issue](https://github.com/thephpleague/flysystem/issues/1787) <- e o que mais me pegou foi a versão da lib, estava usando a primeira versão ainda, fui tentar atualizar, mas precisava atualizar o laravel, atualizei o laravel quebrei foi tudo. Com isso, tive que implementar a lib oficial da AWS para comunicar com S3 sem o ACL.

Mas quando fui subir no ambiente de homologacao, depois de vários minutos esperando a pipeline passar, erro.

Então surgiu, como mockar isso localmente? Para não passar por isso novamente? Ai que surgiu o Localstack, uma ferramenta maravilha que simula produtos da AWS, não todos, mas os mais utilizados sim, com isso vim aqui falar sobre ele, como subir, como usar.

Como Subir Localmente?
---
Bom, basicamente sou adepto sempre a docker, então já recorri a documentacão
- https://docs.localstack.cloud/aws/getting-started/installation/#docker-compose

```
services:
  localstack:
    container_name: "${LOCALSTACK_DOCKER_NAME:-localstack-main}"
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"            # LocalStack Gateway
      - "127.0.0.1:4510-4559:4510-4559"  # external services port range
    environment:
      # LocalStack configuration: https://docs.localstack.cloud/references/configuration/
      - DEBUG=${DEBUG:-0}
    volumes:
      - "${LOCALSTACK_VOLUME_DIR:-./volume}:/var/lib/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

Roda aquele famoso `docker-compose up -d --build`

{{< alert warning "Observacao" >}}
Não sei quando você está lendo isso, mas estamos em Agosto de 2025, muitas coisas podem ter acontecido, podem ter alterados.
{{< /alert >}}

Subiu, agora dá o `docker exec -it localstack-main sh`, vai estar dando attach no container, agora para criar o bucket basta realizar o comando `awslocal s3api create-bucket --bucket sample-bucket`, o retorno esperado deve ser:
```
{
    "Location": "/sample-bucket"
}
```
Para conferir de foi de fato criado, rode o comando `awslocal s3api list-buckets` no próprio attach do container, o retorno deve ser:

```
{
    "Buckets": [
        {
            "Name": "sample-bucket",
            "CreationDate": "2025-08-14T19:03:41.000Z",
            "BucketRegion": "us-east-1"
        }
    ],
    "Owner": {
        "DisplayName": "webfile",
        "ID": "75aa57f09aa0c8caeab4f8c24e99d10f8e7faeebf76c078efc7c6caea54ba06a"
    },
    "Prefix": null
}
```

Basicamente é isso, você localmente nesse momento está rodando um bucket do S3 que já está pronto a ser utilizado, vamos fazer um exemplo básico usando PHP com a lib oficial da AWS.
- https://github.com/aws/aws-sdk-php

Então estou usando [Laravel](https://laravel.com/)

Criei um UploadController:
```php
<?php

namespace App\Http\Controllers;

use Aws\S3\S3Client;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class UploadController extends Controller
{
    public function upload(Request $request): JsonResponse
    {
        $file = $request->file('file');
        $bucket = $request->input('bucket');
        $path = $request->input('path');

        $s3 = new S3Client([
            'version' => 'latest',
            'region' => 'us-east-1',
            'endpoint' => 'http://localstack-main:4566',
            'use_path_style_endpoint' => true,
        ]);

        $s3->putObject([
            'Bucket' => $bucket,
            'Key' => $path . $file->getClientOriginalName(),
            'Body' => $file->getContent(),
        ]);

        return response()->json(['message' => 'File uploaded successfully'], 200);
    }
}
```

Certo, fiz a requisicao, deu 200, mas como valido que realmente deu certo?

Com uma requisicao no seu navegador consegue validar
É uma composicao de valores

`http://localhost:4566/{bucket}/{path}+{nome_arquivo}`

Minha request foi

bucket: sample-bucket
path: teste/
nome_arquivo: teste.png

logo: localhost:4566/sample-bucket/teste/teste.png

Resultado: Baixou o arquivo no meu navegador

