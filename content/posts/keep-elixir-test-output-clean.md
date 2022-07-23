---
title: Keep Elixir test output clean
date: 2022-07-23T11:00:00+02:00
draft: false
tags: ["elixir", "tests"]
---

Does your Elixir test output look like this?

![dirty tests output](/tests-output.png)

If so, keep reading.

It is common to use the Logger library, to log things that happen in your application.

```elixir
  def query_api(url) do
    with {:ok, %{status_code: 200, body: body}} <- HTTPoison.get(url),
         {:ok, json} <- Jason.decode(body) do
      {:ok, json}
    else
      e ->
        Logger.error("impossible to query api: #{inspect(e)}")
        {:error, "api service unavailable"}
    end
  end
```

It is also common to test the behavior of your application, in different scenarios. Some of them generating logs.


```elixir
test "api call with bad url" do
  assert {:error, "api service unavailable"} = "ooops" |> query_api()
end
```

Writing the test this way pollutes your test output with red, even if everything is working as expected, which can be misleading. Or making it more difficult to spot a real problem.

![output of successful test with error logs](/red-output.png)


Here are some solutions:

### Capture logs for all tests
If you want a quick way to make everything green again, you could just go to your `test_helper.exs` file and add the following option:

```elixir
ExUnit.start(capture_log: true)
```

That's efficient, but not very selective. You risk swallowing useful logs showing something is wrong in your code.

### Capture logs for a specific file
You can do the same at the test level module:

```elixir
defmodule MyApp.Query.Test do
  use ExUnit.Case

  @moduletag capture_log: true
  #...
end
```

That's more selective, but you still risk hiding problems.

### Capture logs for a specific function call
My favorite way to do it, using the `ExUnit.CaptureLog.with_log()` [function](https://hexdocs.pm/ex_unit/1.13/ExUnit.CaptureLog.html#with_log/2). You can capture logs only where you expect the logs, and you can assert their content, to be sure everything worked as expected.

```elixir
import ExUnit.CaptureLog

test "api call with bad url" do
  {result, log} = with_log(fn -> "ooops" |> query_api() end)
  
  assert {:error, "api service unavailable"} = result
  assert log =~ "impossible to query api"
end
```

Just be careful, as explained in the doc, `with_log` can also capture logs of another process if you run your tests asynchronously. This is why I asserted the captured `log` contains a string (with `=~`) but didn't check for equality.

After cleaning all the tests output on a project with hundreds of tests, we noticed that a specific test was randomly logging an error, a highly suspect behavior. We wouldn't have spotted the problem with red and yellow messages all over the place. But had we just silenced all the logs at the project level, the issue would have stayed undetected as well. 

This is why I like the `with_log` way better.