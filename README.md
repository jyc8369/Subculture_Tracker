# Subculture Tracker

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="demo/source/app-icon-black.webp" height="240">
  <source media="(prefers-color-scheme: light)" srcset="demo/source/app-icon-black.webp" height="240">
  <img alt="Subculture Tracker" src="demo/source/app-icon-white.webp">
</picture>

Subculture Tracker automatically collects gacha history from supported games, saves it in a shared JSON format, and lets you review the results in a desktop window.

It is designed to work with minimal setup:

- Automatically extracts raw history data and converts it into a common JSON format
- Runs the web interface in a desktop window
- Lets you load and manage records without manually copying data

<img height="180" src="demo/source/app-home.png" alt="Main page preview" /> <img height="180" src="demo/source/app-demo.gif" alt="App demo" />

## Supported Games

- Wuthering Waves
- Arknights Endfield

See [GAME.md](GAME.md) for game-specific notes and limitations.

## Installation

1. Open the [Releases](https://github.com/jyc8369/subculture_tracker/releases) page and download the latest release.
2. Download the asset named like:

```txt
subculture_tracker_X.X.X.zip
```

3. Extract the archive and run `SCT.exe` on Windows, or `SCT` on other supported setups.

## How to Use

1. Launch the game you want to track.
2. Open the in-game gacha history or record screen.
3. Open Subculture Tracker and choose the running game.
4. Click `setting`, enter your profile name, then click `Update/Create Data`.
5. After the data is generated, use the `Select profile file` list at the bottom to load the saved records.

## Data Format

Results are stored locally on your PC as JSON files.

Examples:

- `wuwa_<profilename>.json`
- `endfield_<profilename>.json`
- `web-setting.json`

## FAQ

<details>
<summary>Will this get me banned?</summary>

Subculture Tracker does not modify game files or manipulate memory.

It works by reading WebView-related information from the game logs and using the same history page the game itself uses.

Based on development testing, no account restrictions or warnings have been observed so far.

That said, game policies can change at any time, and you are responsible for your own usage.

</details>

<details>
<summary>How does it work?</summary>

Most supported games provide a way to view gacha history.

Some of them display that history through an in-game WebView. The game logs include the WebView URL or related information, and Subculture Tracker automatically finds it, opens the record endpoint, and converts the results into a readable format.

</details>

<details>
<summary>Will more games be supported?</summary>

Possibly. Each game has different systems, terminology, and storage behavior, so adding support requires manual analysis and testing.

</details>

<details>
<summary>What if a game does not use WebView?</summary>

The current approach depends on WebView-based history pages. Other methods, such as reading memory directly, would carry a much higher risk and are not planned at the moment.

</details>

<details>
<summary>I got an error.</summary>

Please report it on [GitHub Issues](https://github.com/jyc8369/subculture_tracker/issues) and include:

- What you were trying to do
- What happened
- The `lastlog.log` file

</details>

<details>
<summary>Where is the data stored?</summary>

All collected data is stored locally on your machine. Nothing is uploaded to an external server by the app.

</details>

<details>
<summary>It says the game data could not be extracted.</summary>

Check the following:

- The in-game history screen is open
- The game is installed in a standard location

Common causes:

- The URL or token expired over time for some games
- See [GAME.md](GAME.md) for more details

</details>
