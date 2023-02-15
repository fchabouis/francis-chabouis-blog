---
title: Phoenix LiveDashboard with Content Security Policy (CSP)
date: 2023-01-17T11:00:00+02:00
draft: false
tags: ["elixir", "phoenix"]
---

If your Phoenix application enforces CSP rules, and you try to deploy the Phoenix LiveDashboard in production, you will probably get something like this:

![phoenix LiveDashboard without css](/dashboard_no_css.png)

In my case, inline CSS is not loaded because of the [style-src CSP rule](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/style-src) I had to enforce on the project:

```
style-src 'self';
```

This means that all unsafe inline CSS code is disabled by the browser. Unfortunately, the Phoenix LiveDashboard uses inline CSS, and that's not something I can change.

## Nonce basics

The LiveDashboard gives you a way to solve the problem with "Nonces". 

I didn't know about them before, so here is what I learned: a Nonce is a random string
designed to be used **only once**. 

When the server sends the HTTP response back to the user browser, it generates for each request a random string, called a Nonce. For the sake of example, let's say that the generated value is `xyz`.

The Nonce is put in the HTML response at two places:
* In the CSP headers.

For example, to allow safe inline CSS, the CSP header will contain:

```
style-src 'self' 'nonce-xyz';
```

* In the html, on the element that contains inlined CSS I want to allow.
  
```html
<style nonce="xyz">#processes-total-progress{width:0.2%}</style>
```

When the browser receives such a response, it keeps only the inlined CSS matching the Nonce specified in the CSP headers, as it shows that the CSS code
 is genuine and has not been injected by someone else. More info [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/style-src#unsafe_inline_styles).

 ## Phoenix LiveDashboard
 Now that we know what we want to achieve, here is how to do it.

 ### Nonce in the CSP

 There are different ways to add CSP headers to your requests. On the project I work on, I have created a simple **Plug** like this:

 ```elixir
defmodule MyApp.Plugs.CustomSecureBrowserHeaders do

  def init(options), do: options

  def call(conn, _opts) do
    csp_headers = csp_headers(Application.fetch_env!(:my_app, :app_env))

    conn
    |> Phoenix.Controller.put_secure_browser_headers(csp_headers)
  end

  def csp_headers(app_env, nonce) do
    csp_content =
      case app_env do
        :production ->
          """
          style-src 'self';
          """
        _ ->
          nil
      end

    case csp_content do
      nil -> %{}
      csp_content -> %{"content-security-policy" => csp_content |> String.replace("\n", "")}
    end
  end
end

 ```
 
 I use the `put_secure_browser_headers` function provided by Phoenix, with additional CSP headers. So I need to modify the Plug to create the random value and add it to the CSP content. I also register the generated Nonce value in a connection `assign`.

 ```elixir
defmodule MyApp.Plugs.CustomSecureBrowserHeaders do
  def init(options), do: options

  defp generate_nonce(size \\ 10), do: size |> :crypto.strong_rand_bytes() |> Base.url_encode64(padding: false)

  def call(conn, _opts) do
    # a random string is generated
    nonce = generate_nonce()
    csp_headers = csp_headers(Application.fetch_env!(:my_app, :app_env), nonce)

    conn
    # the nonce is saved in the connection assigns
    |> Plug.Conn.assign(:csp_nonce_value, nonce)
    |> Phoenix.Controller.put_secure_browser_headers(csp_headers)
  end

  def csp_headers(app_env, nonce) do
    csp_content =
      case app_env do
        :production ->
          # the nonce is put in the CSP header content
          """
          style-src 'self' 'nonce-#{nonce}';
          """

        _ ->
          nil
      end

    case csp_content do
      nil -> %{}
      csp_content -> %{"content-security-policy" => csp_content |> String.replace("\n", "")}
    end
  end
end

 ```

 ### LiveDashboard
In the Plug above, the Nonce value is assigned to the key `:csp_nonce_value`. I need to pass the information to the Dashboard, and that's it!
In your Router, use the `csp_nonce_assign_key` option:

 ```elixir
live_dashboard("/phoenix-dashboard",csp_nonce_assign_key: :csp_nonce_value)
 ```

 You should now be able to see a Nonce added to your server responses, both in the headers and in the HTML page. Every time you refresh the page, the Nonce value should change.

![phoenix LiveDashboard with CSS](/dashboard_with_css.png)

### Edit
Following a question in the comments, if you want to use a different nonce for style, script and images, you need to make the following changes:

In the Plug code, create 3 assigns containing the 3 different values.
```elixir
  def call(conn, _opts) do
    # a random string is generated
    nonce_1 = generate_nonce()
    nonce_2 = generate_nonce()
    nonce_3 = generate_nonce()

    csp_headers = csp_headers(Application.fetch_env!(:my_app, :app_env), nonce_1, nonce_2, nonce_3)

    conn
    # the nonce is saved in the connection assigns
    |> Plug.Conn.assign(:img_src_nonce, nonce_1)
    |> Plug.Conn.assign(:img_style_nonce, nonce_2)
    |> Plug.Conn.assign(:img_script_nonce, nonce_3)
    |> Phoenix.Controller.put_secure_browser_headers(csp_headers)
  end
```

In the router file, you pass the name of those assigns as options, not the nonce value itself
```elixir
live_dashboard("/phoenix-dashboard",
  metrics: Transport.PhoenixDashboardTelemetry,
  csp_nonce_assign_key: %{
    img: :img_src_nonce,
    style: :img_style_nonce,
    script: :img_script_nonce,
  })
```

The doc is not very clear about what you should pass, but it says you need pass a `%{optional(:img) => atom(), optional(:script) => atom(), optional(:style) => atom()}`. It expects an `atom`, not a string!
🎉