# aws-lamda-logging

1. Você deve criar um dynamodb contendo a "aws_request_id" como chave de partição e "FunctionName" como chave de classificação, ela vai ser a nossa TABELA_EVENTOS.

2. Você deve criar uma função que vai alimentar essa tabela de maneira coerente, ela vai ser nosso EVENT_SAVE. 
```python
import json
import boto3

def lambda_handler(event, context):
    # Recebe um aws_request_id, nome da função e um evento e salva em TABELA_EVENTOS
    if {x for x in event.keys()} != {'aws_request_id', 'FunctionName', 'event'}:
        return {'statusCode':401}
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('eventtable')
    response = table.put_item(
       Item=event
    )
    return response
```
O exemplo acima é em Python, mas você é livre para alterar o que você quiser nisso.
Atribua a essa função o privilégio de ler e escrever no dynamodb.

3. Em TODAS as funções lambdas que você criar, defina essa função ou algo semelhante na sua linguagem de preferência:
```python
import boto3
import sys

client = boto3.client('lambda')
def invoke(function, params, rid):	
	response = client.invoke(
		FunctionName = function,
        InvocationType = 'RequestResponse',
		Payload = json.dumps(params)
        )
	result = json.loads(response['Payload'].read())    
	temp = client.invoke(
		FunctionName = 'EVENT_SAVE',
        InvocationType = 'RequestResponse',
		Payload = json.dumps({
			'response' : result,
			'source':  rid,
			'aws_request_id': response.get('ResponseMetadata').get('RequestId'),
			'FunctionName': function,
			'params': params
	}))
	return result
```

O importante é você enviar para o SALVA_EVENTOS esses dados principais.

5. Usando a API Gateway, crie um recurso "consulta_logs" com um método GET e um POST. No GET ele deve ser capaz de a partir do aws_request_id enviar todas as informações a respeito daquele registro e no POST ele irá receber um aws_request_id e a informação acerca da corretude daquele id. Outro recurso que deve ser adicionado é um "reexecuta_função".

6. Crie uma função "reexecuta_função" e associe ao "reexecuta_função" da API do passo anterior. que deverá ter o seguinte código
```python
import json
import boto3

table = boto3.resource('dynamodb').Table('eventtable3')

def lambda_handler(event, context):
    AWS_REQUEST_ID = event.get('aws_request_id')
    FUNCTION_NAME = event.get('FunctionName')
    VALIDITY = event.get('validity')
    #return event;
    key = {'aws_request_id': AWS_REQUEST_ID, "FunctionName": FUNCTION_NAME}
    print(key)
    response = table.update_item(
        Key=key,
        UpdateExpression='SET validity = :validity',
        ExpressionAttributeValues={
            ':validity': VALIDITY
        },
        ReturnValues="UPDATED_NEW"
    )
    return 
    {
        'statusCode': 200,
        'body': response
    }
```
8. .

