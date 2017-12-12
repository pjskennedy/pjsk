---
layout: post
title:  "Bernie Flanders"
date:   2017-12-09T19:00:00-08:00
categories:
---

I'm new to Elixir, and when learning any new programming language it's best to have a project.

### The Goal

Create a twitter bot that tweets a [Ned Flanders](https://en.wikipedia.org/wiki/Ned_Flanders) version of all of Bernie Sanders' tweets in real time. See the code [here](https://github.com/pjskennedy/bernieflanders).

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Last night, trillions of dollars were stolen from the American people. Wel-diddly-ell, I have news for <a href="https://twitter.com/SenateMajLdr?ref_src=twsrc%5Etfw">@SenateMajLdr</a> and <a href="https://twitter.com/SpeakerRyan?ref_src=twsrc%5Etfw">@SpeakerRyan</a>: You are not going to get away with it. <a href="https://twitter.com/hashtag/TaxScamBill?src=hash&amp;ref_src=twsrc%5Etfw">#TaxScamBill</a></p>&mdash; Bernie Flanders (@bernflanders) <a href="https://twitter.com/bernflanders/status/937017627435614209?ref_src=twsrc%5Etfw">December 2, 2017</a></blockquote>

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Last night, trillions of dollars were stolen from the American people. Well, I have news for <a href="https://twitter.com/SenateMajLdr?ref_src=twsrc%5Etfw">@SenateMajLdr</a> and <a href="https://twitter.com/SpeakerRyan?ref_src=twsrc%5Etfw">@SpeakerRyan</a>: You are not going to get away with it. <a href="https://twitter.com/hashtag/TaxScamBill?src=hash&amp;ref_src=twsrc%5Etfw">#TaxScamBill</a></p>&mdash; Bernie Sanders (@SenSanders) <a href="https://twitter.com/SenSanders/status/937017626039119872?ref_src=twsrc%5Etfw">December 2, 2017</a></blockquote>

### The Translator

Before diving into the application itself, I need to write a "Ned Flanders" translator module in Elixir. Looking at common patterns in Ned's voice I came up with a series of regular expressions as a first attempt.

```elixir
defmodule Bernieflanders.Translator do
  def regexReplace(text, locationRegex, replacement) do
    Regex.replace(locationRegex, text, replacement)
  end

  def translate(text) do
    text
    |> regexReplace(~r/\bhi\b/iu, "hi diddly ho")
    |> regexReplace(~r/\bhello\b/iu, "hi diddly ho")
    |> regexReplace(~r/\bde([a-zA-Z]+)/iu, "de-diddly-\\0")
    |> regexReplace(~r/\b([a-zA-Z]?)el([a-zA-Z]+)/iu, "\\1el-diddly-el\\2")
    |> regexReplace(~r/\br([a-zA-Z]+)/iu, "riddly-r\\1")
    |> regexReplace(~r/([a-zA-Z]+)ility\b/iu, "\\1ilitydility")
    |> regexReplace(~r/([a-zA-Z]+)oos\b/iu, "\\1oosies")
    |> regexReplace(~r/([a-zA-Z]+)b\b/iu, "\\1berino")
    |> regexReplace(~r/([a-zA-Z]+)ch\b/iu, "\\1charoo")
    |> regexReplace(~r/([a-zA-Z]+)bour\b/iu, "\\1bourino")
    |> regexReplace(~r/([a-zA-Z]+)cks\b/iu, "\\1ckeroos")
    |> regexReplace(~r/([a-zA-Z]+)er\b/iu, "\\1eroo")
    |> regexReplace(~r/([a-zA-Z]+)eed\b/iu, "\\1eedily-noodily")
    |> regexReplace(~r/\bgood\b/iu, "dandy")
    |> regexReplace(~r/\bman\b/iu, "fella")
    |> regexReplace(~r/\bstupid\b/iu, "silly")
  end
end
```

This looks confusing, but you can see that it's mostly replacing words that start with _"de"_ to start with _"de-diddly-de"_. Ie, _"democrats"_ is replaced with _"de-diddly-democrats"_, suddenly it sounds much more friendly..

### The Streamer

I then had to interface this with Twitter's API. Twitter has a great streaming API where you can stream tweets given a set of criteria. Given Elixir's pipe-operator it was easy to accomplish all of this:

1. Stream tweets from Twitter related to Bernie Sanders
2. Filter the tweets to those written by Bernie (no re-tweets or replies)
3. Fetch tweet from REST API (for reasons beyond me, twitter has decided to truncate tweets in the streaming API)
4. Translate tweet text to be from Ned Flanders
5. Tweet to Parody account ([@bernflanders](https://twitter.com/bernflanders))

```elixir
defmodule Bernieflanders.Twitterstream do
  def stream(handles) do
    IO.puts "Starting Twitter Stream"

    userIds = twitterIds(handles)

    # Stream tweets from given user
    ExTwitter.stream_filter(follow: Enum.join(userIds, ","))
    # Filter tweets
    |> Stream.filter(&(filterTweet(&1, userIds)))
    # Expand tweets with REST Api
    |> Stream.map(&(ExTwitter.show(&1.id, tweet_mode: "extended")))
    # Translate Tweet to Ned Flanders
    |> Stream.map(&(Bernieflanders.Translator.translate(&1.full_text)))
    # Truncate Tweet to fit Twitter's Rules
    |> Stream.map(&(String.slice(&1, 0..278)))
    # Tweet
    |> Stream.map(&(ExTwitter.update(&1)))
    |> Enum.to_list
  end

  defp twitterIds(handles) do
    handles
    |> Enum.map(&(ExTwitter.user(&1)))
    |> Enum.map(&(&1.id))
  end

  # Remove retweets
  defp filterTweet(%{text: "RT" <> _}, _), do: false

  # Remove mentions and replies
  defp filterTweet(%{user: %{id: id}}, userIds), do: Enum.member?(userIds, id)

  # Remove anything else
  defp filterTweet(_, _), do: false
end
```

### The Application

Surely, this module isn't perfect, but following Elixir's "Let it fail" methodology, let's wrap it in a [GenServer](https://hexdocs.pm/elixir/GenServer.html) and construct a supervisor to re-start the process if it fails.

```elixir
defmodule Bernieflanders.Server do
  use GenServer

  def start_link(list) do
    GenServer.start_link(__MODULE__, list, name: __MODULE__)
  end

  def start_link do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(handles) do
    # Create a new process to stream tweets
    spawn_link(Bernieflanders.Twitterstream, :stream, [handles])
    {:ok, handles}
  end
end

defmodule Bernieflanders do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    # Load Conifg
    :ok = ExTwitter.configure(Application.get_env(:extwitter, :oauth))
    # Get twitter handles to stream
    handles = Application.get_env(:bernieflanders, :handles)

    # Construct workers to supervise
    children = [
      worker(Bernieflanders.Server, [handles])
    ]

    # Start supervisor
    opts = [strategy: :one_for_one, name: Bernieflanders.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

<a class="twitter-timeline" href="https://twitter.com/bernflanders">Tweets by @bernflanders</a>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
