# APS HTTP Bridge

### APS/HTTP

APS与HTTP服务的桥接，为现有的HTTP服务提供APS的接口。简单描述为，以URL作为方法名，QUERY_STRING作为参数，返回结果采用JSON。

#### REQUEST

  * **Envelop**:  
    HTTP头信息`APS-ENVELOP` (octets)
  * **Sequence**:  
    HTTP头信息`APS-SEQUENCE` (octets)
  * **Timestamp**:  
    HTTP头信息`APS-TIMESTAMP` (itoa)
  * **Expiry**:  
    HTTP头信息`APS-EXPIRY` (itoa)
  * **Method**:  
    Request URI，以'/'开头。可能去掉前缀
  * **Params**:  
    * 当HTTP的方法是GET参数使用QueryString
    * 当以`application/x-www-form-urlencoded`方式POST时，使用Form的值
    * 当以`application/json`方式POST时，参数以JSON方式存在body中
    * 当以`application/x-magpack`方式POST时，参数以MessagePack方式存在body中
    * 不支持其他Content-Type，例如`multipart/form-data`
  * **Extras**:  
    HTTP头信息`APS-X-...`保存扩展信息

#### REPLY

  * **Envelop**:  
    HTTP头信息`APS-ENVELOP` (octets)，原封不动复制请求的值
  * **Sequence**:  
    HTTP头信息`APS-SEQUENCE` (octets)，原封不动复制请求的值
  * **Timestamp**:  
    HTTP头信息`APS-TIMESTAMP` (itoa)
  * **Status**:  
    `Status`
  * **Reply**:  
    * 当以请求是以`application/x-magpack`方式POST时，结果以MessagePack方式存在body中
    * 否则，结果以`application/x-magpack`方式存在body中
  * **Extras**:  
    HTTP头信息`APS-X-...`保存扩展信息

### APS/FastCGI

这是为了让APS桥接器直接与php-fpm通信，而不需要经过nginx。

#### REQUEST

  * **Envelop**:  
    `HTTP_APS_ENVELOP` (octets)
  * **Sequence**:  
    `HTTP_APS_SEQUENCE` (octets)
  * **Timestamp**:  
    `HTTP_APS_TIMESTAMP` (itoa)
  * **Expiry**:  
    `HTTP_APS_EXPIRY` (itoa)
  * **Method**:  
    Request URI，以'/'开头。可能去掉前缀
  * **Params**:  
    * 当`REQUEST_METHOD`是`GET`时，参数使用`QUERY_STRING`
    * 当`CONTENT_TYPE`是`application/x-www-form-urlencoded`时，使用Form值
    * 当`CONTENT_TYPE`是`application/json`时，参数以JSON方式存在body中
    * 当`CONTENT_TYPE`是`application/x-magpack`时，参数以MessagePack方式存在body中
    * 不支持其他`CONTENT_TYPE`，例如`multipart/form-data`
  * **Extras**:  
    `HTTP_APS_X_...`保存扩展信息

#### REPLY

处理方式与APS/HTTP一致