---
title: "File Downloading in Headless Chrome Using ChromeDriver and Hound"
date: 2019-09-16T13:10:45+07:00
categories: ["Elixir"]
tags: ["Headless Chrome", "ChromeDriver", "Hound"]
draft: false
---

Recently, I am making a simple Elixir application performing some actions to a website in an automated way.

The automated testing tool is a perfect candidate to be used to help to build application like that.
I use [Hound](https://github.com/HashNuke/hound) as browser automation library
and Chrome as a controlled browser.
For the browser driver, I use [ChromeDriver](https://chromedriver.chromium.org/).

# Problem

Back to building my application.

One of the tasks my application doing is downloading a file on the website.
That is not a problem in a normal setup. However, the file is not downloaded when
the headless mode is enabled.

<!--more-->

After Google has enlightened me, in security perspective, that behaviour is
needed to prevent
malicious website quietly download unwanted files through the browser in headless mode.

# Solution

For the solution, we have to instruct ChromeDriver via REST API to allow file downloading  

```text
POST http://localhost:9515/session/<session_id>/chromium/send_command
```

```json
{
    "cmd": "Page.setDownloadBehavior",
    "params": {
        "behavior": "allow",
        "downloadPath": "/path/download"
    }

}
```

_Note:_  `9515` is ChromeDriver default port

## Using Hound

Unfortunately, Hound does not provide `send_command` as its API method. But, we
can use `Hound.RequestUtils.make_req` to send API request to ChromeDriver.

For the complete demonstration, these are steps to build a sample application that
download file (`Docs.zip`) in <https://elixir-lang.org/docs.html>

### Chrome & ChromeDriver Setup

* Download and install Chrome
* Download and install
  [ChromeDriver](https://chromedriver.chromium.org/downloads). Make sure Chrome
  and ChromeDriver have same major version.
* Start ChromeDriver and leave it running:
  
  ```shell
  $ chromedriver --verbose
    Starting ChromeDriver 77.0.3865.40 (f484704e052e0b556f8030b65b953dce96503217-refs/branch-heads/3865@{#442}) on port 9515
    Only local connections are allowed.
    Please protect ports used by ChromeDriver and related test frameworks to prevent access by malicious code.
  ```

### Building Application

Generate a new application

```shell
$ mix new file_downloader
$ cd file_downloader
```

Add `hound` as a dependency library

```elixir
# file_downloader/mix.exs
defmodule FileDownloader.MixProject do
  use Mix.Project

  def project do
    [
      app: :file_downloader,
      version: "0.1.0",
      elixir: "~> 1.9",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  def application do
    [
      extra_applications: [:logger]
    ]
  end

  defp deps do
    [
      {:hound, "~> 1.1.0"} # <- add hound library
    ]
  end
end
```

Download the dependencies

```shell
$ mix deps.gets
```

Config `hound` to use ChromeDriver and Chrome in headless mode

```elixir
# file_downloader/config/config.exs
use Mix.Config

config :hound, driver: "chrome_driver", browser: "chrome_headless"
```

Code the logic of our application. Step 3 describes how to enable file downloading in
headless mode.

```elixir
# file_downloader/lib/file_downloader.ex
defmodule FileDownloader do
  use Hound.Helpers
  import Hound.RequestUtils

  def download_elixir_docs do
    # 1) Start hound session
    Hound.start_session()

    # 2) Visit the website
    navigate_to("https://elixir-lang.org/docs.html")

    # 3) By using 'Hound.RequestUtils.make_req', enable file downloading
    {:ok, download_path} = File.cwd()
    session_id = Hound.current_session_id()

    make_req(
      :post,
      "session/#{session_id}/chromium/send_command",
      %{
        cmd: "Page.setDownloadBehavior",
        params: %{behavior: "allow", downloadPath: download_path}
      }
    )

    # 4) Find download link and click it to download file
    download_link = {:xpath, "//*[@id='stable']/small/a"}
    download_link |> click()

    # 5) Wait until download process is completed
    wait_download_started(download_path)
    wait_download_completed(download_path)

    # 6) Stop hound session
    Hound.end_session()
  end

  defp wait_download_started(download_path) do
    wait_crdownload(download_path, true)
  end

  defp wait_download_completed(download_path) do
    wait_crdownload(download_path, false)
  end

  defp wait_crdownload(dir, exist?, wait_time \\ 1000) do
    count_crdownload =
      dir
      |> Path.join("*.crdownload")
      |> Path.wildcard()
      |> Enum.count()

    unless((count_crdownload != 0 && exist?) || (count_crdownload == 0 && !exist?)) do
      Process.sleep(wait_time)
      wait_crdownload(dir, exist?, wait_time)
    end
  end
end
```

Run the application and wait until it is finished.

```shell
$ mix run -e FileDownloader.download_elixir_docs
```

The downloaded file (`Docs.zip`) will be available in the current directory (`file_downloader`).

### Troubleshooting

If you got a runtime error (invalid session id) like this : 

```shell
$ mix run -e FileDownloader.download_elixir_docs
Compiling 1 file (.ex)
** (RuntimeError) invalid session id
    (hound) lib/hound/request_utils.ex:52: Hound.RequestUtils.handle_response/3
    (file_downloader) lib/file_downloader.ex:13: FileDownloader.download_elixir_docs/0
    (stdlib) erl_eval.erl:680: :erl_eval.do_apply/6
    (elixir) lib/code.ex:240: Code.eval_string/3
    (elixir) lib/enum.ex:783: Enum."-each/2-lists^foreach/1-0-"/2
    (elixir) lib/enum.ex:783: Enum.each/2
    (mix) lib/mix/tasks/run.ex:141: Mix.Tasks.Run.run/5
```

it might be caused by different versions of Chrome and ChromeDriver.
You can check the log of running ChromeDriver.

```shell
$ chromedriver --verbose
...

[1578222297.224][INFO]: Failed to connect to Chrome. Attempting to kill it.
[1578222297.244][INFO]: [e5d2ce77a1b56643db3d11b6fad7d946] RESPONSE InitSession ERROR session not created: This version of ChromeDriver only supports Chrome version 77

...
```
