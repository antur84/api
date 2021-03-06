# Cinode API
Hello, fellow developer! We're glad you're here and interested in our API. We have put together some samples for you to get started! If you have any issues or question regarding the technically around the API please open an issue here on GitHub and we will try to help you. The API is still in Beta, so please bear with us. If you have other questions about our services please contact us via our regular support channels.

## API Account
In order to connect to our token endpoint you need to create an API account (AccessId and AccessSecret). You can do this under `/account` when logged in. If you can't find it, please contact our support for activation. This will be used instead of your ordinary username and password. Every request made to the API will be made in the context of the owner of the API account and will need certain access rights to access different parts of the API.

## Rate limiting
The rate limit time interval window is 2 seconds and the limit is according to the current plan set up for each company. On each request there will be response headers indicating how much you have used in the current window and how many requests are remaning. There will also be headers to indicate how many daily requests there is left.

### Plans
#### Token
| Daily requests | Rate limit | Rate limit reset |
| -------------- | ---------- | ---------------- |
| 10000          | 2          | 2                |

### HTTP Headers and error response codes

#### Headers
* `x-ratelimit-limit` - The maximum number of requests current `AccessId` or `CompanyId` can perform per time interval window.
* `x-ratelimit-remaining` - The number of requests left for the time interval window. 
* `x-ratelimit-reset` - The remaining window before the rate limit resets. This will be presented in seconds.
* `x-route-daily-requests-left` - Indicates how many requests you can still make for this route during the ongoing day (24 hrs UTC).
* `x-daily-requests-left` - Indicates how many requests you can still make during the ongoing day (24 hrs UTC).

After one successfull request the response headers will look like this
```
x-ratelimit-limit: 40
x-ratelimit-remaining: 39
x-ratelimit-reset: 2
x-daily-requests-left: 9999
```

#### Error response codes
If you exceed the rate limit, our API will start rejecting your request and you will be receiving a error response with status code `HTTP 429 "Too many requests"` and with the following example body
```JSON
{"description":"Request limit was exceeded, please try again after 2 seconds from now","moreInfo":"https://github.com/Cinode-Labs/api#rate-limiting","status":429}
```

### Daily usage
The daily usage is counted per Company. The `x-daily-requests-left` in the response header will indicate how many requests you can still make during the ongoing day. Some endpoints have a special rate limit and the remaining requests is indicated in the `x-route-daily-requests-left` response header. The daily limit will be reset at midnight in UTC.

## Retrieve access_token
To access our API endpoints you need to exchange your AccessId and AccessSecret to an `access_token`. The access token is in `jwt` format and have certain properties regarding subject id, certain claims and validity.

To retrieve access and refresh token make a `GET` request to `https://api.cinode.com/token` with `Authentication: Basic` in the header. AccessId and AccessSecret must be in the following format "accessId:accessSecret" and be `base64`. In return you will get a `json` object with `access_token` and `refresh_token`.

Example using `HttpClient` in C#
```C#
private static HttpClient httpClient = new HttpClient();
var accessId = "[ACCESSID]";
var accessSecret = "[ACCESSSECRET]";
var basicParameter = Convert.ToBase64String(Encoding.UTF8.GetBytes($"{accessId}:{accessSecret}"));

httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic", basicParameter);

var basicResponse = await httpClient.GetAsync("https://api.cinode.com/token");

if (basicResponse.IsSuccessStatusCode)
{
    var basicResult = await basicResponse.Content.ReadAsStringAsync();

    var definition = new { access_token = "", refresh_token="" };
    var tokenResponse = JsonConvert.DeserializeAnonymousType(basicResult, definition);
}
```

Response
```JSON
{"access_token": "[access_token]", "refresh_token":"[refresh_token]"}
```

The `access_token` is valid for `120 seconds`, after that you can use the `refresh_token` to retrieve a new one.

## Refresh token
When the `access_token` is expired you can either retrieve a new `access_token` and `refresh_token` via the `/token` endpoint or you can `POST` the `refresh_token` to `https://api.cinode.com/token/refresh` with the following payload.
```JSON
{"refreshToken": "[refresh_token]"}
```

Example using `HttpClient` in C#
```C#
private static HttpClient httpClient = new HttpClient();
var refreshToken ="[REFRESHTOKEN]";

var request = new HttpRequestMessage(HttpMethod.Post, new Uri("https://api.cinode.com/token/refresh"));
var requestContent = new {refreshToken = refresh_token};

request.Content = new StringContent(JsonConvert.SerializeObject(requestContent),
Encoding.UTF8,
"application/json");//CONTENT-TYPE header;

var refreshResponse = await _httpClient.SendAsync(request);
if (refreshResponse.IsSuccessStatusCode)
{
    var refreshResult = await refreshResponse.Content.ReadAsStringAsync();
    var definition = new { access_token = "", refresh_token="" };
    var tokenResponse = JsonConvert.DeserializeAnonymousType(refreshResult, definition);
}
```
Response
```JSON
{"access_token": "[access_token]", "refresh_token":"[refresh_token]"}
```

## Authenticate against the API
For every call to our API endpoint you need to provide the `access_token` as a `Bearer` in the `Authorization` header.
```
Authorization: Bearer access_token
```

Example using `HttpClient` in C#
```C#
private static HttpClient httpClient = new HttpClient();
httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", tokenResponse.access_token);
httpClient.DefaultRequestHeaders.Accept.Clear();
httpClient.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

var response = await httpClient.GetAsync(url);

if (response.IsSuccessStatusCode)
{
    var result = await response.Content.ReadAsStringAsync();
}
```

## PHP
Here is an example how to use `PHP` to retrieve an `access_token` and use it to `GET` active Users and list their first name.

```PHP
<?php

#Change these.
$companyId = [COMPANYID];
$accessId = [ACCESSID]
$accessSecret = [ACCESSSECRET];


#Dont't change anything below here!
$apiBaseUrl ='https://api.cinode.com';
$tokenUrl = $apiBaseUrl.'/token';
$usersUrl = $apiBaseUrl.'/v0.1/companies/'.$companyId.'/users/';

$auth = base64_encode($accessId.':'.$accessSecret);
$authContext = stream_context_create(['http' => ['header' => "Authorization: Basic $auth"]]);
$tokenResponse = file_get_contents($tokenUrl, false, $authContext );
$tokenObj = json_decode($tokenResponse);
$access_token = $tokenObj->access_token;
$context = stream_context_create(["http" => ["method" => "GET", "header" => "Accept: application/json\r\n" ."Authorization: Bearer $access_token\r\n"]]);
$usersResponse = file_get_contents($usersUrl, false, $context);
$users = json_decode($usersResponse);
foreach ($users as $user){
        echo $user->firstName;
}
?>
```