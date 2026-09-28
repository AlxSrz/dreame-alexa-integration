# Puente AWS Lambda (Alexa Smart Home → Home Assistant)

Función que recibe la directiva de Alexa y la reenvía a
`${BASE_URL}/api/alexa/smart_home` con el token del account linking.

- **Función:** `dreame-alexa-bridge` (Python 3.12, `us-east-1`, 128 MB, timeout 15 s)
- **Rol:** `dreame-alexa-lambda-role` (`AWSLambdaBasicExecutionRole`)
- **ARN:** `arn:aws:lambda:us-east-1:283449825232:function:dreame-alexa-bridge`
- **Variable de entorno:** `BASE_URL=https://ha.alexa.alce-soft.com`
- **Código:** `lambda_function.py` (gist oficial de Home Assistant,
  https://gist.github.com/matt2005/744b5ef548cc13d88d0569eea65f5e5b)

## ⚠️ Gotcha crítico: el `Principal` correcto es el de *connectedhome*

Alexa usa **dos principals distintos** y es fácil equivocarse (nos pasó):

| Principal | Sirve para |
|---|---|
| `alexa-appkit.amazon.com` | Skills de **conversación / custom** (ASK) |
| `alexa-connectedhome.amazon.com` | Skills **Smart Home** ← este proyecto |

Con el principal equivocado, la consola de Alexa Developer rechaza el ARN al
guardar el *Default endpoint* con un mensaje que despista, porque **sí ve un
permiso**, pero del tipo de evento equivocado:

> *"Please make sure that 'Alexa Smart Home' is selected for the event source
> type"*

Más gente tropieza con esto porque SAM arrastra el mismo defecto: su evento
`AlexaSkill` solo concede el principal de appkit
(https://github.com/awslabs/serverless-application-model/issues/613).

## Comandos de despliegue / corrección

Todo con la cuenta personal de AWS (`283449825232`) y su perfil local
(`--profile personal-dreame`; las credenciales viven en `../.env`, que está
en `.gitignore`).

```bash
# Permiso de invocación correcto (Smart Home), acotado al Skill ID
aws lambda add-permission \
  --function-name dreame-alexa-bridge \
  --statement-id alexa-smart-home \
  --action lambda:InvokeFunction \
  --principal alexa-connectedhome.amazon.com \
  --event-source-token amzn1.ask.skill.20c9a886-3f35-4f7d-bb4d-b4bedd909fe0 \
  --region us-east-1 --profile personal-dreame

# Verificar (ojo: en la salida cruda el campo Policy viene como string con
# JSON escapado, hay que desescaparlo para leerlo cómodo)
aws lambda get-policy --function-name dreame-alexa-bridge \
  --region us-east-1 --profile personal-dreame

# Actualizar código tras editar lambda_function.py
python3 -c "import zipfile; zipfile.ZipFile('/tmp/f.zip','w',zipfile.ZIP_DEFLATED).write('lambda_function.py','lambda_function.py')"
aws lambda update-function-code --function-name dreame-alexa-bridge \
  --zip-file fileb:///tmp/f.zip --region us-east-1 --profile personal-dreame
```

El usuario IAM usado (`dreame-lambda-setup`) tiene permisos acotados a Lambda
y al rol de esta función; si hace falta `lambda:RemovePermission` u otro, se
agrega a mano en IAM (no lo incluimos por defecto para no pedir de más).
