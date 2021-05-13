---
layout: post
title:  How to implement U2F Authentication with Phoenix LiveView
description: Arguably, the strongest multi-factor authentication method is U2F. Here is how to integrate it with your Phoenix LiveView application.
date:   2021-05-13 08:01:35 +0200
tags:   [development, u2f, phoenix, elixir]
---

If you want to secure your users' accounts beyond simple username and password authentication, [U2F](https://developers.yubico.com/U2F/) is arguably the most secure multi-factor authentication method you can offer. In order to log in or register, your user needs to authenticate using a physical device like a [YubiKey](https://www.yubico.com/). After entering the username and password, the user needs to authorize the physical device to accept and solve a challenge given to it from your application. If the device answers correctly to the challenge, the user is logged in successfully. If the user does not have the correct physical devise, she won't be able to log in.

By having a physical device as a multi-factor, you can prevent unauthorized access to a user's account even if an attacker gained access to the user's username and password. Without the physical device, the attacker won't be able to log in, end of story. The U2F-method secures accounts with such ease and speediness that [Google rolled out U2P-devices](https://www.yubico.com/resources/reference-customers/google/) to all their employees.

![Demo of registering and logging in a user with a YubiKey 5](/assets/gifs/u2f-demo.gif)

So, all this sounds cool, but how can you add a U2F authentication method to your [Phoenix LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html) project? Here is how:

## The Setup

Let's first create a new Phoenix LiveView project. You can skip this step if you have an existing project already. Create a new project with:

```bash
mix phx.new app --live
```

This command will create a new Phoenix LiveView project. Next, let us include the [u2f_ex](https://github.com/Ianleeclark/u2f_ex) library, which handles registering the U2F-devices and creation of challenges when a user wants to log in. In your `mix.exs`, add the following dependency:

```elixir
def deps do
  [
    ...
    {:u2f_ex, "~> 0.4.2"}
  ]
end
```

Fetch the dependency with `mix deps.get`. Now, you have a project and the necessary dependency, so let's create the database schemas to register the U2F-devices.

## Enabling SSL

U2F requires an SSL connection and does not allow communication over an unprotected HTTP-connection. Offering a HTTPS-connection is typically not problem in production since we would enable SSL encryption anyway. However, we want to test the U2F authentication locally, which means we have set up a HTTPS-connection to our server locally. We can do that by running the following command to create a new self-signed SSL certificate:

```bash
mix phx.gen.cert
```

This will create a self-signed SSL certificate and store it at `priv/cert/selfsigned_key.pem`. Now, enable the SSL endpoint in `dev.exs` like this:

```elixir
# config/dev.exs

config :app, AppWeb.Endpoint,
  ...
  http: [port: 4000],
  # Add the following lines
  https: [
    port: 4001,
    cipher_suite: :strong,
    keyfile: "priv/cert/selfsigned_key.pem",
    certfile: "priv/cert/selfsigned.pem"
  ]
```

You can now access your server through [https://localhost:4001](https://localhost:4001). If your browser warns you about an *insecure* or *invalid* certificate, consider allowing insecure certificates for [localhost](https://stackoverflow.com/a/31900210/4138739).

## The Schema for the U2F-Keys

We want to save the U2f-keys given to us by the users for creating challenges for them whenever they try to log in. In your `/lib/app` folder, create a new folder called `u2f` and add the following schema for the U2F-keys:

```elixir
# /lib/u2f/u2f_key.ex

defmodule App.U2FKey do
  use Ecto.Schema
  import Ecto.Changeset

  schema "u2f_keys" do
    field(:public_key, :string)
    field(:key_handle, :string)
    field(:version, :string, size: 10, default: "U2F_V2")
    field(:app_id, :string)

    # Change the following field if you want to connect the U2F-key
    # to a user in a different way (e.g. through a 'user_id' instead)
    field(:username, :string)

    timestamps()
  end

  @doc false
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:public_key, :key_handle, :version, :app_id, :username])
    |> validate_required([
      :public_key, 
      :key_handle, 
      :version, 
      :app_id, 
      :username
    ])
  end
end
```

This schema will save all necessary information about the U2F-key. I added a `username` field to fetch the correct U2F-key for the user who wants to log in. If you want to establish a relationship between a user and a U2F-key in a different way, feel free to change this field to e.g. `user_id` instead .

Next, let's create a migration for this schema with:

```bash
mix ecto.gen.migration create_u2f_keys
```

Open the newly create migration and add the following code to it:

```elixir
defmodule App.Repo.Migrations.CreateU2fKey do
  use Ecto.Migration

  def change do
    create table(:u2f_keys) do
      add(:public_key, :string)
      add(:key_handle, :string)
      add(:version, :string, size: 10, default: "U2F_V2")
      add(:app_id, :string)

      # Again, change this field to e.g. user_id if you want to
      add(:username, :string)

      timestamps()
    end
  end
end
```

Now, execute the migration with `mix ecto.migrate`. This command will create the `u2f_keys` table in your database.

## The Context for the U2F-keys

For fetching and creating U2F-keys, the `u2f_ex`-library requires us to create a context that implements the `U2FEx.PKIStorageBehaviour` behaviour, so let's create a `pki_storage.ex` file and add the following code to it:

```elixir
# /lib/app/u2f/pki_storage.ex

defmodule App.PKIStorage do
  import Ecto.Query

  alias App.Repo
  alias App.U2FKey

  alias U2FEx.PKIStorageBehaviour

  @behaviour U2FEx.PKIStorageBehaviour

  @impl PKIStorageBehaviour
  def list_key_handles_for_user(username) do
    keys =
      from(u in U2FKey,
        where: u.username == ^username,
        select: map(u, [:version, :key_handle, :app_id])
      )
      |> Repo.all()

    {:ok, keys}
  end

  @impl PKIStorageBehaviour
  def get_public_key_for_user(username, key_handle) do
    from(u in U2FKey,
      where: u.username == ^username and u.key_handle == ^key_handle
    )
    |> Repo.one()
    |> case do
      nil -> {:error, :public_key_not_found}
      %U2FKey{public_key: public_key} -> {:ok, public_key}
    end
  end

  def create_u2f_key(username, %U2FEx.KeyMetadata{} = key_metadata) do
    attrs = Map.merge(Map.from_struct(key_metadata), %{username: username})

    %U2FKey{}
    |> U2FKey.changeset(attrs)
    |> Repo.insert()
  end
end
```

The `PKIStorage`-module allows us to fetch the `key_handles` and the `public_key` for a user. It also offers a `create_u2f_key/2`-method to register a new U2F-key for a user. Once again, please change the `username` parameter to e.g. `user_id`, if you establish the connection between `u2f_key` and `user` differently.

## The Templates for Authentication

Let's add a simple template, which offers two forms: One form for registering a U2F-Key and one form for logging in with the U2F-key. For brevity, I put both forms into a single template. Feel free to separate them in your project.

```eex
# /lib/app_web/live/page_live.html.leex

# Register a U2F-key using a Phoenix LiveView
<section class="phx-hero">
  <%= f = form_for :registration, "#", [id: "registration-form", phx_submit: "start_registration", phx_hook: "Registration"] %>
    <%= text_input f, :username, placeholder: "Choose a username..." %>
    <%= submit "Register" %>
  </form>
</section>

# Login with a U2F-Key using a Phoenix Controller
<section class="phx-hero">
  <%= f = form_for :login, "#", [id: "login-form", phx_submit: "start_login", phx_hook: "Login"] %>
    <%= text_input f, :username, placeholder: "Enter your username..." %>
    <%= submit "Login" %>
  </form>
</section>
```

The forms above allow us to register and log in. Let's create two more templates that show the result of the login attempt.

```eex
# /lib/app_web/templates/login/success.html.eex
<section>
  <h1>Success!</h1>
  <h3>You're logged in as: <%= @username %></h3>
</section>

# /lib/app_web/templates/login/failure.html.eex
<section>
  <h1>Error!</h1>
  <h3>Could not log you in:</h3>
  <h2><%= @error %></h2>
</section>
```

## The LiveView for Authentication

Now, let's get to the interesting parts. First, we need to write a LiveView that handles registering a U2F-device and subsequently logging in with it. I will paste the entire module here first and then discuss its functions:

```elixir
# /lib/app_web/live/page_live.ex

defmodule AppWeb.PageLive do
  use AppWeb, :live_view

  require Logger

  alias App.{PKIStorage, U2FKey}
  alias U2FEx.KeyMetadata

  @impl true
  def mount(_params, _session, socket) do
    {:ok, socket}
  end

  @impl true
  def handle_event(
        "start_registration",
        %{
          "registration" => %{"username" => username}
        },
        socket
      ) do
    {:ok, %{registerRequests: register_requests}} =
      U2FEx.start_registration(username)

    socket =
      socket
      |> assign(:username, username)
      |> push_event("register", %{registerRequests: register_requests})

    {:noreply, socket}
  end

  @impl true
  def handle_event(
        "finish_registration",
        %{"response" => device_response},
        %{assigns: %{username: username}} = socket
      ) do
    {:ok, %KeyMetadata{} = key_metadata} =
      U2FEx.finish_registration(username, device_response)
    
    {:ok, %U2FKey{} = key} = 
      PKIStorage.create_u2f_key(username, key_metadata)

    {:noreply, put_flash(socket, :success, "Registration successful.")}
  end

  @impl true
  def handle_event(
        "start_login",
        %{
          "login" => %{"username" => username}
        },
        socket
      ) do
    {:ok, %{challenge: challenge, registeredKeys: registered_keys}} =
      U2FEx.start_authentication(username)

    sign_requests = Enum.map(
      registered_keys, 
      &Map.merge(&1, %{challenge: challenge})
    )

    socket =
      socket
      |> assign(:username, username)
      |> push_event("login", %{signRequests: sign_requests})

    {:noreply, socket}
  end

  @impl true
  def handle_event(
        "finish_login",
        %{"response" => device_response},
        %{assigns: %{username: username}} = socket
      ) do
    encoded_response = Jason.encode!(device_response)
    U2FEx.finish_authentication(username, encoded_response)
    |> case do
      :ok ->
        token = Phoenix.Token.sign(AppWeb.Endpoint, "username", username)
        callback_path = Routes.login_path(socket, :login, token)
        {:noreply, redirect(socket, to: callback_path)}

      error ->
        Logger.error(inspect(error))
        {:noreply, put_flash(socket, :error, "Login failed.")}
    end
  end
end
```

The first functions of the LiveView handle the registration of a new U2F-key. 

#### Registering a U2F Device

User can register their U2F-device with a username and call the `start_registration`-handler. The handler simply passes on the `username` to the `U2FEx`-library, which creates a challenge for the U2F-device and returns `registration_requests`. We assign the currently registering `username` to the socket and push the `registration_requests` back to the JavaScript-Hooks, which I will explain later.

Once a user has authorized the U2F-device to answer the `registration_requests` by pressing a button on the device or touching it, the JavaScript-Hooks will send a `finish_registration`-event back to the LiveView. The LiveView receives the device response for the presented challenges and passes on the `username` and `device_response` to the `U2FEx`-library. The library verifies the response against the challenge and returns metadata for the U2F-key. We store the metadata in the database by calling the `PKIStorage.create_u2f_key/2`-function.

The user has now successfully registered a new U2F-device for the given `username`.

#### Log in with a U2F Device

The login flow using a U2F-device is similar to the registration flow. A user first sends a login request to the LiveView and provides a `username` for which a U2F-device was registered previously. We pass the `username` into the `U2FEx.start_authentication/1` function, which fetches the registered U2F-key metadata from the database and creates a `login-challenge`. We merge every `registered_key` with the `challenge` since this is what the JavaScript-library handles the frontend communication requires. Eventually, we push the `sign_requests` to the JavaScript-Hooks.

Next, the JavaScript-Hooks present the challenge to the U2F-device, the user authorizes the U2F-device to answer the challenge, and eventually the LiveView receives the `finish_login`-event, which includes the device response. We present the response to the `U2FEx`-library as `encoded json` and let the library verify the challenge. If the library returns a success, we know that the user owns the previously registered U2F-device and we can log her in. Unfortunately, we cannot edit the user's session from inside the LiveView. We need to redirect to a `Phoenix.Controller` instead, which then uses `put_session/3`, to authenticate the user for subsequent requests. More on this later.

## The JavaScript-Hooks for Authentication

Before we write our Hooks, please download the [u2f-api.js](https://github.com/fido-alliance/google-u2f-ref-code/blob/master/u2f-gae-demo/war/js/u2f-api.js)-library, which handles the communication with the U2F-device in the frontend. Save its code into `/assets/js/u2f-api.js`.

Now, let's create our LiveView-Hooks. Create the file `/assets/js/u2f.js` and paste the following code into it:

```javascript
// /assets/js/u2f.js

import u2fApi from "u2f-api";

const Registration = {
  mounted() {
    this.handleEvent("register", ({ registerRequests }) => {
      u2fApi.register(registerRequests).then((deviceResponse) => {
        this.pushEvent("finish_registration", { response: deviceResponse });
      });
    });
  },
};

const Login = {
  mounted() {
    this.handleEvent("login", ({ signRequests }) => {
      u2fApi.sign(signRequests).then((deviceResponse) => {
        this.pushEvent("finish_login", { response: deviceResponse });
      });
    });
  },
};

export { Registration, Login };
```

As you can see, the Hooks are minimal. They receive the `register` and `login`-events from the LiveView, pass on the challenges to the `u2fApi`, and push the device response back to the LiveView. For brevity, I left out most of the error handling in the Hooks. Thus, in your project, you probably want to handle any errors in the `.then()`-callbacks.

Don't forget to add these Hooks to your LiveSocket in `app.js`:

```javascript
// /assets/js/app.js

import { Registration, Login } from "./u2f";

var Hooks = {};
Hooks.Registration = Registration;
Hooks.Login = Login;

...

let liveSocket = new LiveSocket("/live", Socket, {
  params: { _csrf_token: csrfToken },
  hooks: Hooks,
});
```

## Authenticating the User in the Phoenix.Session

After user log in successfully, we redirect them from our LiveView to a `LoginController` in order to modify the users' [Plug.Session](https://hexdocs.pm/plug/Plug.Session.html). Let's recall the code:

```elixir
U2FEx.finish_authentication(username, encoded_response)
|> case do
  :ok ->
    token = Phoenix.Token.sign(AppWeb.Endpoint, "username", username)
    callback_path = Routes.login_path(socket, :login, token: token)
    {:noreply, redirect(socket, to: callback_path)}
  error ->
    ...
```

We want to tell the `LoginController` that the user logged in successfully and use a [Phoenix.Token](https://hexdocs.pm/phoenix/Phoenix.Token.html) for this purpose. We create a token after the successful login by signing the username with the secret of our `AppWeb.Endpoint`. We then redirect to the `login-callback`-route and add the `token` as a query parameter.

Now, let's have a look at the `LoginController`, which receives the request:

```elixir
# /lib/app_web/controller/login_controller.ex

defmodule AppWeb.LoginController do
  use AppWeb, :controller

  def login(conn, %{"token" => token}) do
    Phoenix.Token.verify(AppWeb.Endpoint, "username", token, max_age: 60)
    |> case do
      {:ok, username} ->
        conn
        |> put_session(:username, username)
        |> render( "success.html", username: username)

      {:error, error} ->
        render(conn, "failure.html", error: error)
    end
  end
end
```

Also, don't forget to add the `LoginController` to your `router.ex`:

```elixir
# /lib/app_web/router.ex

scope "/", AppWeb do
  pipe_through :browser

  live "/", PageLive, :index
  get "/login-callback", LoginController, :login
end
```

The `LoginController` receives the callback with the `token` as a parameter. It then verifies the token with the secret from the `AppWeb.Endpoint`, a signing salt (`username`), and a `max_age`. In our case, the creation and verification of the token should not take longer than the redirect, so we set the `max_age` to 60 seconds. If you set a higher `max_age` you risk that an attacker can reuse the token to sign in at a later point in time. Thus, keep the `max_age` small.

After the controller successfully verifies the token, it puts the username into the user's session and renders the `success.html` template. On further requests, the user will now provide her `username` in the `session` and is therefore fully authenticated. Hurray!

## Caveats

* The `u2f-api.js` is `archived` by now, but I could not find a better library for communicating with my YubiKey. Google recommends using the [WebAuthn](https://github.com/w3c/webauthn)-API in the future. However, neither Google nor Yubico gives examples of how to integrate with said API. Yubico's tutorials use the [u2f-api.js](https://github.com/fido-alliance/google-u2f-ref-code/blob/master/u2f-gae-demo/war/js/u2f-api.js) library, which is why I decided to use it for this project also. If you know more about which JavaScript-library to use, please contact me on [Twitter](https://twitter.com/pjullrich).

* In your project, don't use the `username` for registering a U2F-device, since others can register a U2F-device for usernames of others as well. Use a `user_id` instead.

* Be aware that you leak the `user_id` to the user's browser if you use `Phoenix.Token` since the tokens are not encrypted. If this is a security issue for you, consider encrypting the `user_id` before redirecting the user to the `login-callback`.




