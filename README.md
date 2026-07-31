# Módulo: s3-secure

Módulo Terraform reutilizável para criar um bucket S3 **seguro por padrão**, consolidando as boas práticas aplicadas manualmente no [lab04-s3](../../lab04-s3) deste repositório.

## O que o módulo faz

- Cria um bucket S3
- Habilita versionamento (configurável)
- Aplica criptografia server-side (`AES256` ou `aws:kms`, configurável)
- Bloqueia todo acesso público (configurável, mas ligado por padrão)
- Aplica bucket policy que **nega qualquer requisição fora de HTTPS** (`aws:SecureTransport = false`)

A filosofia é "secure by default": quem usar o módulo sem passar nenhuma variável opcional já recebe um bucket com versionamento, criptografia, bloqueio público e HTTPS-only ativados.

## Uso básico

```hcl
module "meu_bucket" {
  source = "../../modules/s3-secure"

  bucket_name = "minha-empresa-dados-2026"

  tags = {
    Ambiente = "dev"
    Projeto  = "portfolio"
  }
}
```

## Inputs

| Nome                   | Tipo          | Padrão    | Descrição                                                                 |
|------------------------|---------------|-----------|----------------------------------------------------------------------------|
| `bucket_name`           | `string`      | —         | Nome do bucket (obrigatório, deve ser globalmente único)                  |
| `enable_versioning`     | `bool`        | `true`    | Habilita versionamento de objetos                                         |
| `sse_algorithm`         | `string`      | `"AES256"`| `"AES256"` ou `"aws:kms"`                                                  |
| `kms_key_id`            | `string`      | `null`    | ARN da chave KMS (obrigatório se `sse_algorithm = "aws:kms"`)             |
| `block_public_access`   | `bool`        | `true`    | Bloqueia acesso público no bucket                                         |
| `enforce_https`         | `bool`        | `true`    | Aplica policy negando tráfego fora de HTTPS                               |
| `force_destroy`         | `bool`        | `false`   | Permite destruir bucket mesmo com objetos dentro (útil em teste/ministack)|
| `tags`                  | `map(string)` | `{}`      | Tags aplicadas ao bucket                                                   |

## Outputs

| Nome                          | Descrição                                       |
|--------------------------------|--------------------------------------------------|
| `bucket_id`                    | Nome (ID) do bucket criado                       |
| `bucket_arn`                   | ARN do bucket criado                             |
| `bucket_regional_domain_name`  | Domínio regional do bucket                       |
| `versioning_status`            | Status de versionamento aplicado (`Enabled`/`Suspended`) |

## Testando via ministack

Este módulo não define provider próprio — ele herda o provider configurado no módulo raiz que o chama. Para testar localmente contra o `ministack`, use o mesmo padrão de override manual já usado nos outros labs deste repositório (endpoints customizados apontando para `http://localhost:4566`, `access_key = "test"` / `secret_key = "test"`).

Um exemplo completo de uso está em [`examples/s3-secure-basic`](../../examples/s3-secure-basic).

## Notas técnicas

`force_destroy = false` por padrão é proposital: em um cenário real, um bucket com dados não deveria ser apagável só porque o Terraform pediu `destroy`. Ligamos `force_destroy = true` apenas no exemplo de teste, para facilitar o ciclo de `apply`/`destroy` durante validação no `ministack`.

### Comportamento intermitente observado no `ministack`

Durante a validação deste módulo, um `terraform apply` concluiu com sucesso (5 recursos criados, sem erro), mas uma checagem manual logo em seguida (`aws s3api list-buckets` e `head-bucket`) não encontrou o bucket — sem nenhum log de exclusão no `ministack`. Um `terraform destroy` rodado nesse estado reportou "nada a destruir", como se o Terraform tivesse perdido o rastro do recurso.

Para isolar a causa, cada recurso do módulo (bucket, versionamento, criptografia, public access block, bucket policy) foi testado individualmente via `aws s3api`, de forma sequencial — todos sobreviveram normalmente. Rodando o `apply` do módulo completo novamente, o bucket permaneceu estável (confirmado imediatamente e após 15s de espera), e o `destroy` subsequente funcionou sem problemas.

**Conclusão**: o comportamento não se reproduziu de forma consistente e não foi isolado a nenhum recurso específico do módulo — os testes sequenciais via CLI sugerem que a causa mais provável é uma condição de corrida no `ministack` ao processar as múltiplas chamadas que o provider `hashicorp/aws` dispara em paralelo logo após criar o bucket (o provider busca atributos computados como CORS, ACL, lifecycle, logging, entre outros, mesmo quando o módulo não os utiliza). Isso não foi observado nos labs anteriores deste repositório, que usam recursos AWS mais simples ou sequenciais.

**Recomendação prática**: ao testar este módulo via `ministack`, sempre confirme o estado do recurso com `aws s3api list-buckets` logo após o `apply`, antes de seguir para o `destroy`. Em caso de inconsistência, um novo `apply` normalmente resolve.
