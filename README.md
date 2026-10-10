# N++ Discord Server Analytics

These awesome analytics were generated with [Chat Analytics](https://github.com/mlomb/chat-analytics) using the Discord chats scraped via [DiscordChatExporter](https://github.com/tyrrrz/discordchatexporter), which you can find in JSON format [here](https://mega.nz/file/VcAkkJRa#iRyNzPSNpBIfgCt0ACElNic0fa6qZMPJr_2k4cYhLtU).

You may also be interested in [this SQL archive](https://mega.nz/file/NQJVnZQD#fh-11AgcHrfqifcZ7twq9fn7rKSEmsjs12h8SLezI8s) which contains additional fields dumped with [my DCE fork](https://github.com/edelkas/DiscordChatExporter) and converted from JSON to SQL using [dce2sql](https://github.com/edelkas/dce2sql).

- [How to export chats](#how-to-export-chats)
   * [Extended info](#extended-info)
- [How to generate the analytics](#how-to-generate-the-analytics)
- [How to convert to SQL](#how-to-convert-to-sql)

## How to export chats

Download [DCE](https://github.com/Tyrrrz/DiscordChatExporter/releases/latest) and create an environment variable `DISCORD_TOKEN` with the token of the Discord account to use, preferably a bot to avoid the ban risk. The run the following PowerShell script, which exports each channel and thread to its own JSON file and separates them by year.

```powershell
# Config variables for DCE
$SERVER = 197765375503368192 # N++ server
$FOLDER = ".\exports"
$START  = 2016
$END    = 2026

# Yearly exports
foreach ($year in $START..$END) {
    .\DiscordChatExporter.Cli.exe exportguild -g $SERVER --include-threads All -f Json -o "$FOLDER\$year\[%T] %C.json" --after "$year-01-01T00:00:00+00:00" --before "$year-12-31T23:59:59+00:00" --utc --parallel 4 --markdown false
}
```

This can take a long time, like multiple days, specifically due to Discord's rate limiting, so if you're just interested in the N++ server, you can download a complete JSON dump from September 2026 [here](https://mega.nz/file/VcAkkJRa#iRyNzPSNpBIfgCt0ACElNic0fa6qZMPJr_2k4cYhLtU).

You can also export just the channel list like so:

```powershell
.\DiscordChatExporter.Cli.exe channels -g 197765375503368192 --include-threads All > channels.txt
```

### Extended info

I also have a [DCE fork](https://github.com/edelkas/DiscordChatExporter) that adds plenty of extended information not exported by standard DCE for user data, channel data, guild data, message components, and more. It also [normalizes](https://en.wikipedia.org/wiki/Database_normalization) data to remove redundancy from the JSON outputs, bringing the total size down significantly. It's able to cache results between runs, fixes a few bugs, and more.

You can find an extended dump as of September 2026 [here](https://mega.nz/file/1FBXQAKa#dzPB2CoW_8XsLsb5fPmpw1TROzUQ6fa-ccYjfa5mcYM). Do note that it's no longer compatible with Chat Analytics, though.

With this fork you can also export the entire **user list**, even for those that haven't messages in the server at all, with:

```powershell
.\DiscordChatExporter.Cli.exe exportusers -g 197765375503368192 -o "users.json" --utc
```

This takes only a few seconds because each request fetches 1000 users. You can add `--full-users` to fetch some extra info (e.g. banners), but that requires one request per user, so the time balloons to a few hours.

## How to generate the analytics

To generate the HTML report, download [Chat Analytics](https://github.com/mlomb/chat-analytics/releases/latest) or install it directly with:

```
npm install -g chat-analytics
```

Then navigate to the location of your JSON exports and run:

```
npx chat-analytics -p discord -i "/**/*.json
```

This will generate a single, roughly 50MB, HTML file named `report.html`.

## How to convert to SQL

The JSON exports are good for archival, and compatible with other tools, but not very good for querying or building other tools. I made [dce2sql](https://github.com/edelkas/dce2sql) to convert DCE's JSON exports, including my own extended ones, to a SQL database.

It can be fed arbitrarily many JSON dumps from any number of servers, even multiple files corresponding to the same channel, and it'll update the information without ever removing anything (e.g. messages by deleted users), tracking message edits, and more. It can also recover raw mentions from dumps that excluded / resolved them (due to the `--markdown` flag).

To install it, clone the repo and run `pip install .`, or `pip install '.[all]'` if you need the db drivers. You can find a MySQL dump [here](https://mega.nz/file/NQJVnZQD#fh-11AgcHrfqifcZ7twq9fn7rKSEmsjs12h8SLezI8s).

The dump was fed a complete vanilla dump from April 2025, together with an extended dump from September 2026, and finally the complete user list, via the following script:

```powershell
# 2025 vanilla exports
foreach ($year in 2016..2025) {
  dce2sql -e mysql --user root --password root --unresolve --channels "2025_dump\channels.txt" discord "2025_dump\exports\$year\"
}

# 2026 full user export
dce2sql -e mysql --user root --password root discord "2026_dump\users_full.json"

# 2026 extended normalized raw exports
foreach ($year in 2016..2026) {
  dce2sql -e mysql --user root --password root discord "2026_dump\exports\$year\"
}
```

You'll have to adapt the paths and database configuration variables.