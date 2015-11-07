# oauth-google

web.config:
```xml
<appSettings>
  <add key="GoogleAuth.ClientId" value="YOUR_CLIENT_ID" />
  <add key="GoogleAuth.VerifyUrl" value="https://www.googleapis.com/oauth2/v3/tokeninfo?id_token={token}" />
</appSettings>
```

index.cshtml:
```html
<script src="https://apis.google.com/js/api:client.js"></script>
<script src="/location/to/scripts/oauth-google.js"></script>

<script>
    var clientId = '@ConfigurationManager.AppSettings["GoogleAuth.ClientId"]';
    var validationUrl = '/Account/LoginExternal';
    
    function initExternalLogin() {
        var googleLogin = new GoogleLogin(clientId, validationUrl);
        googleLogin.attachButton(document.getElementById('loginGoogle'));
    }
    
    $(function() {
      initExternalLogin();
    });
</script>

<a href="#" id="loginGoogle">
    Login with Google Account
</a>

```

oauth-google.js:
```js
function GoogleLogin(clientId, validationUrl) {
    var self = this;

    this.validationUrl = validationUrl;
    
    gapi.load('auth2', function () {
        self.auth2 = gapi.auth2.init({
            client_id: clientId,
            cookiepolicy: 'single_host_origin',
            scope: 'email'
        });
    });
}

GoogleLogin.prototype.attachButton = function(element) {
    var self = this;

    $(element).on('click', function () {
        self.auth2.signIn().then(function (googleUser) {
            var user = googleUser.getBasicProfile();

            var request = $.ajax({
                url: self.validationUrl,
                type: 'post',
                data: JSON.stringify({
                    token: googleUser.getAuthResponse().id_token,
                    email: user.getEmail(),
                    name: user.getName(),
                    imageUrl: user.getImageUrl(),
                    provider: 'google'
                }),
                dataType: 'json',
                contentType: 'application/json; charset=utf-8'
            });

            request.done(function (response) {
                console.log('User logged in!');
                window.location.href = response.redirectUrl;
            });
        }, function (message) {
            alert('Error');
            console.log(message);
        });
    });
}
```

AccountsService.cs:
```csharp
public async Task<User> LoginExternal(LoginExternal model)
{
    if (model.Provider == "google")
    {
        await LoginExternalGoogleValidate(model);
    }
    else
    {
        throw new InvalidOperationException("Provider not supported");
    }

    // add user if doesn't exists or get existing
    var user = ...;

    // set auth cookies or generate a token

    return user;
}

private async Task LoginExternalGoogleValidate(LoginExternal model)
{
    using (var httpClient = new HttpClient())
    {
        var googleApiUrl = ConfigurationManager.AppSettings["GoogleAuth.VerifyUrl"]
            .Replace("{token}", model.Token);

        var message = await httpClient.GetAsync(googleApiUrl);

        var result = await message.Content.ReadAsAsync<UserLoginExternalGoogleResult>();

        if (!result.EmailVerified || result.Email != model.Email 
              || result.Audience != ConfigurationManager.AppSettings["GoogleAuth.ClientId"])
        {
            throw new InvalidOperationException("Validation error");
        }
    }
}
```

LoginExternal.cs:
```csharp
public class LoginExternal
{
    public string Provider { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string ImageUrl { get; set; }
    public string Token { get; set; }
}
```

LoginExternalModel.cs
```csharp
public class LoginExternalModel
{
    public string Provider { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string ImageUrl { get; set; }
    public string Token { get; set; }
}

```

UserLoginExternalGoogleResult.cs:
```csharp
public class UserLoginExternalGoogleResult
{
    public string Email { get; set; }

    [JsonProperty("email_verified")]
    public bool EmailVerified { get; set; }

    [JsonProperty("aud")]
    public string Audience { get; set; }
}
```

AccountController.cs:
```csharp
public async Task<JsonResult> LoginExternal(LoginExternalModel model)
{
  var loginData = new LoginExternal
  {
      Name = model.Name,
      Email = model.Email,
      ImageUrl = model.ImageUrl,
      Provider = model.Provider,
      Token = model.Token
  };
  
  User user;

  try
  {
      user = await _accountsService.LoginExternal(loginData);
  }
  catch (Exception)
  {
      // failure
  }

  // success
  return Json(new { redirectUrl: "/" });
}

```
