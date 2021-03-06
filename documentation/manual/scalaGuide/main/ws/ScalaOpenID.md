<!--- Copyright (C) 2009-2013 Typesafe Inc. <http://www.typesafe.com> -->
# OpenID Support in Play

OpenID is a protocol for users to access several services with a single account. As a web developer, you can use OpenID to offer users a way to log in using an account they already have, such as their [Google account](https://developers.google.com/accounts/docs/OpenID). In the enterprise, you may be able to use OpenID to connect to a company’s SSO server.

## The OpenID flow in a nutshell

1. The user gives you his OpenID (a URL).
2. Your server inspects the content behind the URL to produce a URL where you need to redirect the user.
3. The user confirms the authorization on his OpenID provider, and gets redirected back to your server.
4. Your server receives information from that redirect, and checks with the provider that the information is correct.

Step 1 may be omitted if all your users are using the same OpenID provider (for example if you decide to rely completely on Google accounts).

## Usage

To use OpenId, first add `ws`  to your `build.sbt` file:

```scala
libraryDependencies ++= Seq(
  ws
)
```

## OpenID in Play

The OpenID API has two important functions:

* `OpenID.redirectURL` calculates the URL where you should redirect the user. It involves fetching the user's OpenID page asynchronously, this is why it returns a `Future[String]`. If the OpenID is invalid, the returned `Future` will fail.
* `OpenID.verifiedId` needs an implicit `Request` and inspects it to establish the user information, including his verified OpenID. It will do a call to the OpenID server asynchronously to check the authenticity of the information, this is why a `Future[UserInfo]`  is returned. If the information is not correct or if the server check is false (for example if the redirect URL has been forged), the returned `Future` will fail.

If the `Future` fails, you can define a fallback, which redirects back the user to the login page or return a `BadRequest`.

Here is an example of usage (from a controller):

```scala
def login = Action {
  Ok(views.html.login())
}

def loginPost = Action.async { implicit request =>
  Form(single(
    "openid" -> nonEmptyText
  )).bindFromRequest.fold(
    { error =>
      Logger.info("bad request " + error.toString)
      Future.successful(BadRequest(error.toString))
    },
    { openId =>
      OpenID.redirectURL(openId, routes.Application.openIDCallback.absoluteURL())
        .map(url => Redirect(url))
        .recover { case t: Throwable => Redirect(routes.Application.login) }
    }
  )
}

def openIDCallback = Action.async { implicit request =>
  OpenID.verifiedId.map(info => Ok(info.id + "\n" + info.attributes))
    .recover {
      case t: Throwable =>
      // Here you should look at the error, and give feedback to the user
      Redirect(routes.Application.login)
    }
}
```

## Extended Attributes

The OpenID of a user gives you his identity. The protocol also supports getting [extended attributes](http://openid.net/specs/openid-attribute-exchange-1_0.html) such as the e-mail address, the first name, or the last name.

You may request *optional* attributes and/or *required* attributes from the OpenID server. Asking for required attributes means the user cannot login to your service if he doesn’t provides them.

Extended attributes are requested in the redirect URL:

```scala
OpenID.redirectURL(
    openid,
    routes.Application.openIDCallback.absoluteURL(),
    Seq("email" -> "http://schema.openid.net/contact/email")
)
```

Attributes will then be available in the `UserInfo` provided by the OpenID server.

> **Next:** [[OAuth | ScalaOAuth]]
