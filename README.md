# Notas para la Integración del TPV-Virtual de Redsys

Para la integración del medio de pago TPV-Virtual de Redsys, están son mis notas que nos ayudaron en la integración realizada en el marco de un proyecto de consultoría realizado por [Extly TECH](https://extly.com/) y [Extly](https://extly.com/).

Los siguientes datos de ejemplo para pruebas manuales y unitarias automatizadas son independientes de tecnología utilizadas y los incluyo en este documento para facilitar las pruebas futuras.

## Introducción

Para comenzar, la documentación oficial es la referencia más completa del TPV-Virtual de Redsys: `TPV-Virtual Manual Integración - Redirección.pdf`, que puede ser descargado desde el área de desarrolladores: <https://pagosonline.redsys.es/desarrolladores.html>.

En nuestro caso, realizamos la integración en un proyecto en Laravel (PHP) dentro de un CMS Joomla, con las librerías [ssheduardo/redsys-laravel](https://github.com/ssheduardo/redsys-laravel) y [sermepa/sermepa](https://github.com/ssheduardo/sermepa).

Para las pruebas, utilizamos los **Datos Genéricos De Prueba** provistos por Redsys aquí: <https://pagosonline.redsys.es/realizacion-pruebas.html#pruebas-redireccion>

El objetivo de este documento es presentar en un solo lugar y de forma breve las claves para la integración del TPV-Virtual de Redsys:

1. generar el formulario de pago para Redsys, y
1. simular notificaciones de pago para pruebas manuales y testeo automatizado

## Configuración Básica del Sistema

Para la configuración básica del sistema, y generar el formulario de pago TPV-Virtual de Redsys, son necesarios estos datos:

- Tradename
- Merchant Code
- Terminal
- Key SHA-256
- Enviroment

En las siguentes pruebas, utilizaré estos datos:

```
REDSYS_TRADENAME="MI TIENDA"
REDSYS_MERCHANT_CODE=123456789
REDSYS_TERMINAL=1
REDSYS_KEY=AB12ISHAJDcqfhPSZTzpaFDXb8zXvFsA
REDSYS_ENVIROMENT=test
```

## Formulario de Pago

Partiendo de la configuración, es posible generar el siguiente formulario de pago HTML para el TPV-Virtual de Redsys:

```html
<form method="post" id="redsys_form" name="redsys_form"
    action="https://sis-t.redsys.es:25443/sis/realizarPago">

    <input type="hidden" name="Ds_MerchantParameters"
        value="ewogICAiRFNfTUVSQ0hBTlRfTUVSQ0hBTlRDT0RFIjoiMTIzNDU2Nzg5IiwKICAgIkRTX01FUkNIQU5UX1RFUk1JTkFMIjoiMSIsCiAgICJEU19NRVJDSEFOVF9UUkFOU0FDVElPTlRZUEUiOiIwIiwKICAgIkRTX01FUkNIQU5UX0FNT1VOVCI6MzAwMDAsCiAgICJEU19NRVJDSEFOVF9DVVJSRU5DWSI6Ijk3OCIsCiAgICJEU19NRVJDSEFOVF9PUkRFUiI6IjAwMDc1RTEwNjREMiIsCiAgICJEU19NRVJDSEFOVF9QUk9EVUNUREVTQ1JJUFRJT04iOiJNaSBQcm9kdWN0byIsCiAgICJEU19NRVJDSEFOVF9NRVJDSEFOVFVSTCI6Imh0dHA6XC9cL215LnNpdGVcL2luZGV4LnBocD9vcHRpb249Y29tX21pY29tcG9uZW50ZSZ0YXNrPXJlZHN5c1BheW1lbnRDYWxsYmFjayIsCiAgICJEU19NRVJDSEFOVF9NRVJDSEFOVE5BTUUiOiJNSSBUSUVOREEiCn0=">

    <input type="hidden" name="Ds_Signature"
        value="1RfAiAxOy3tACIAmgEQbIu1WZ1BZuuvSQpOFTixPeTU=">

    <input type="hidden" name="Ds_SignatureVersion"
        value="HMAC_SHA256_V1">

    <a class="boton">Pagar</a>
</form>
```

Los datos incluídos en el campo `Ds_MerchantParameters` son los siguientes:

```json
{
   "DS_MERCHANT_MERCHANTCODE":"123456789",
   "DS_MERCHANT_TERMINAL":"1",
   "DS_MERCHANT_TRANSACTIONTYPE":"0",
   "DS_MERCHANT_AMOUNT":30000,
   "DS_MERCHANT_CURRENCY":"978",
   "DS_MERCHANT_ORDER":"00075E1064D2",
   "DS_MERCHANT_PRODUCTDESCRIPTION":"Mi Prpoducto",
   "DS_MERCHANT_MERCHANTURL":"http:\/\/my.site\/index.php?option=com_micomponente&task=redsysPaymentCallback",
   "DS_MERCHANT_MERCHANTNAME":"MI TIENDA"
}
```

**Atención** al parámetro `DS_MERCHANT_MERCHANTURL`, ya que es la URL que recibirá el POST de notificación con el resultado de pago.

## Inicialización del Service Provider la App Laravel

En particular para la inicialización del Service Provider la App Laravel:

```php
$rootUri = rtrim(\JURI::root(), '/');
$urlNotification = $rootUri.'/index.php?index.php?option=com_micomponente&task=redsysPaymentCallback';

$app['config']['redsys'] = [
    'tradename' => Cms::getSetting('REDSYS_TRADENAME'),
    'merchantcode' => Cms::getSetting('REDSYS_MERCHANT_CODE'),
    'terminal' => Cms::getSetting('REDSYS_TERMINAL', '1'),
    'key' => Cms::getSetting('REDSYS_KEY'),
    'enviroment' => Cms::getSetting('REDSYS_ENVIROMENT', 'test'),

    // Atención al Parámetro url_notification que recibirá el POST de notificación
    'url_notification' => $urlNotification,

    'url_ok' => Cms::getSetting('REDSYS_URL_OK'),
    'url_ko' => Cms::getSetting('REDSYS_URL_KO'),
];
```

## Notificaciones de Pago de Ejemplo para el TPV-Virtual de Redsys

Para probar el código del sistema, es conveniente poder generar los comandos de prueba para facilitar el testing manual y automatizado en pruebas unitarias. Para el caso anterior, estas son el testeo a realizar:

### Caso Tarjeta Aceptada

```json
{
   "Ds_Date":"02%2F01%2F2020",
   "Ds_Hour":"17%3A49",
   "Ds_SecurePayment":"1",
   "Ds_Amount":"30000",
   "Ds_Currency":"978",
   "Ds_Order":"00025E0E1EA8",
   "Ds_MerchantCode":"285298006",
   "Ds_Terminal":"002",
   "Ds_Response":"0000",
   "Ds_TransactionType":"0",
   "Ds_MerchantData":"",
   "Ds_AuthorisationCode":"016900",
   "Ds_ConsumerLanguage":"1",
   "Ds_Card_Country":"724",
   "Ds_Card_Brand":"1"
}
```

```bash
curl -XPOST "http://my.site/index.php?index.php?option=com_micomponente&task=redsysPaymentCallback" \
    -d 'Ds_SignatureVersion=HMAC_SHA256_V1' \
    -d 'Ds_MerchantParameters=eyJEc19EYXRlIjoiMDIlMkYwMSUyRjIwMjAiLCJEc19Ib3VyIjoiMTclM0E0OSIsIkRzX1NlY3VyZVBheW1lbnQiOiIxIiwiRHNfQW1vdW50IjoiMzAwMDAiLCJEc19DdXJyZW5jeSI6Ijk3OCIsIkRzX09yZGVyIjoiMDAwMjVFMEUxRUE4IiwiRHNfTWVyY2hhbnRDb2RlIjoiMjg1Mjk4MDA2IiwiRHNfVGVybWluYWwiOiIwMDIiLCJEc19SZXNwb25zZSI6IjAwMDAiLCJEc19UcmFuc2FjdGlvblR5cGUiOiIwIiwiRHNfTWVyY2hhbnREYXRhIjoiIiwiRHNfQXV0aG9yaXNhdGlvbkNvZGUiOiIwMTY5MDAiLCJEc19Db25zdW1lckxhbmd1YWdlIjoiMSIsIkRzX0NhcmRfQ291bnRyeSI6IjcyNCIsIkRzX0NhcmRfQnJhbmQiOiIxIn0=' \
    -d 'Ds_Signature=qtbG1sMxshyh4YJ_284dPNm2YYUahNVbNMm4HZeQbKg='
```

### Caso Tarjeta Rechazada

```json
{
   "Ds_Date":"02%2F01%2F2020",
   "Ds_Hour":"17%3A53",
   "Ds_SecurePayment":"0",
   "Ds_Amount":"30000",
   "Ds_Currency":"978",
   "Ds_Order":"00025E0E1FD5",
   "Ds_MerchantCode":"285298006",
   "Ds_Terminal":"002",
   "Ds_Response":"9551",
   "Ds_TransactionType":"0",
   "Ds_MerchantData":"",
   "Ds_AuthorisationCode":"++++++",
   "Ds_ConsumerLanguage":"1",
   "Ds_Card_Country":"724"
}
```

```bash
curl -XPOST "http://my.site/index.php?index.php?option=com_micomponente&task=redsysPaymentCallback" \
    -d 'Ds_SignatureVersion=HMAC_SHA256_V1' \
    -d 'Ds_MerchantParameters=eyJEc19EYXRlIjoiMDIlMkYwMSUyRjIwMjAiLCJEc19Ib3VyIjoiMTclM0E1MyIsIkRzX1NlY3VyZVBheW1lbnQiOiIwIiwiRHNfQW1vdW50IjoiMzAwMDAiLCJEc19DdXJyZW5jeSI6Ijk3OCIsIkRzX09yZGVyIjoiMDAwMjVFMEUxRkQ1IiwiRHNfTWVyY2hhbnRDb2RlIjoiMjg1Mjk4MDA2IiwiRHNfVGVybWluYWwiOiIwMDIiLCJEc19SZXNwb25zZSI6Ijk1NTEiLCJEc19UcmFuc2FjdGlvblR5cGUiOiIwIiwiRHNfTWVyY2hhbnREYXRhIjoiIiwiRHNfQXV0aG9yaXNhdGlvbkNvZGUiOiIrKysrKysiLCJEc19Db25zdW1lckxhbmd1YWdlIjoiMSIsIkRzX0NhcmRfQ291bnRyeSI6IjcyNCJ9' \
    -d 'Ds_Signature=dzZNeQNwgJaxpkidAHLHydmzYCnbV6JWvwfqngXCUOE='
```

### Herramientas Utilizadas

Para las pruebas y ensayos:

- Base64 Encode and Decode - Online - <https://www.base64encode.org>
- JSON Formatter & Validator - <https://jsonformatter.curiousconcept.com>
- HMAC Generator / Tester Tool - <https://www.freeformatter.com/hmac-generator.html#ad-output>

Para generar las firmas de los mensajes, este es el código PHP que utiliza el paquete [ssheduardo/sermepa](https://github.com/ssheduardo/sermepa/blob/master/src/Sermepa/Tpv/Tpv.php#L391):

```php
    /**
     * Generate Merchant Signature Notification
     *
     * @param string $key
     * @param string $data
     *
     * @return string
     */
    public function generateMerchantSignatureNotification($key, $data)
    {
        $key = $this->decodeBase64($key);

        // Decode data base64
        $decode = $this->base64_url_decode($data);

        // Los datos decodificados se pasan al array de datos
        $parameters = $this->JsonToArray($decode);
        $order = $this->getOrderNotification($parameters);
        $key = $this->encrypt_3DES($order, $key);

        // Generated Hmac256 of Merchant Parameter
        $result = $this->hmac256($data, $key);

        return $this->base64_url_encode($result);
    }
```

## Conclusión

Con los puntos anteriores es posible generar y probar el workflow de pago completo del TPV-Virtual de Redsys, logrando generar el formulario de pago para Redsys, y simular notificaciones de pago para pruebas manuales y testeo automatizado.

## Copyright & Licencia

- Copyright (c)2012-2020 Extly, CB. All rights reserved.
- Creative Commons - [Atribución/Reconocimiento-CompartirIgual 4.0 Internacional](https://creativecommons.org/licenses/by-sa/4.0/legalcode.es)
- Creative Commons - [Attribution-ShareAlike 4.0 International
](https://creativecommons.org/licenses/by-sa/4.0/legalcode)
