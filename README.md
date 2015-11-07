# oauth-google

web.config:
```xml
<appSettings>
  <add key="GoogleAuth.ClientId" value="YOUR_CLIENT_ID" />
  <add key="GoogleAuth.VerifyUrl" value="https://www.googleapis.com/oauth2/v3/tokeninfo?id_token={token}" />
</appSettings>
```

index.html:
```html
<script src="https://apis.google.com/js/api:client.js"></script>

<script>
    function initExternalLogin() {
        var googleLogin = new GoogleLogin('@ConfigurationManager.AppSettings["GoogleAuth.ClientId"]', '/Account/LoginExternal');
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
        // Retrieve the singleton for the GoogleAuth library and set up the client.
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
                ajaxRequestSuccess(response);
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

    // login here using cookies/token

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

        if (!result.EmailVerified || result.Email != model.Email ||
            result.Audience != ConfigurationManager.AppSettings["GoogleAuth.ClientId"])
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

    public string InvitationToken { get; set; }
}

```

AccountController.cs:
```csharp
public async Task<JsonResult> LoginExternal(LoginExternalModel model)
{
  User user;
  
  var loginData = new LoginExternal
  {
      Name = model.Name,
      Email = model.Email,
      ImageUrl = model.ImageUrl,
      Provider = model.Provider,
      Token = model.Token
  };
  
  try
  {
      user = await _membershipService.LoginExternal(loginData);
  }
  catch (Exception)
  {
      return this.AjaxError();
  }

  // some extra logic

  return this.JsonRedirect(successUrl);
}

```
