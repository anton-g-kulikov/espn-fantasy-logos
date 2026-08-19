# ESPN Fantasy Logos

Public image assets and an unofficial implementation guide for custom ESPN Fantasy Hockey team logos.

ESPN's current interfaces do not expose custom-logo controls consistently, but the Fantasy Hockey team-write API still accepts a public image URL. The procedure below documents the exact workflow verified against the 2027 season.

> This relies on undocumented ESPN endpoints and may stop working if ESPN changes its clients or APIs. Never commit or share ESPN authentication cookies.

## Hosted logos

### League 1069086703

| Team ID | Logo | CDN URL |
| --- | --- | --- |
| 5 | Valencia Colden Nuts (current) | `https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/1069086703/team-5-valencia-colden-nuts.jpg` |
| 5 | Ice Floyds (previous) | `https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/1069086703/team-5-ice-floyds.jpg` |

### League 1615029211

| Team ID | Logo | CDN URL |
| --- | --- | --- |
| 4 | Pavlovo-Posdskie Ynitazi | `https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/1615029211/team-4-ynitazi.jpg` |
| 5 | Dimas's Hipsters | `https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/1615029211/team-5-dimas-hipsters.jpg` |
| 10 | Zorros — Another One Bites the Ice | `https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/1615029211/team-10-zorros.jpg` |

## How the workaround works

ESPN maintains at least two copies of fantasy-team presentation data:

1. The league/team record served by `lm-api-reads.fantasy.espn.com`.
2. A delegated fantasy preference served by `fan.api.espn.com` and used during client hydration.

Posting a custom URL to the team-write API updates the league record. ESPN normalizes the requested `CUSTOM` type to `CUSTOM_VALID`. Saving a real team-name change in the ESPN Fantasy mobile app synchronizes the delegated preference.

Do not write directly to the fan-preference endpoint. ESPN treats fantasy logo fields as delegated: direct requests may be accepted while silently discarding the logo, or may restore an older shield and team name to the league record.

The team header initially uses the custom URL as a CSS background and also loads it through a hidden `<img>`. If that image request errors, ESPN's handler replaces the custom image with its default shield. Stale browser state can therefore produce a brief custom-logo flash followed by the shield even when both API records are correct.

## Requirements

- A public HTTPS image URL.
- Square artwork, prepared as a 450 × 450 JPEG at approximately 75% quality.
- A file below 500 KB, matching ESPN's [published logo limit](https://support.espn.com/hc/en-us/articles/115003860672-Adding-a-Team-Logo).
- `curl`, `jq`, Git, and—on macOS—`sips`.
- An ESPN account that owns/manages the target team, is a second manager of that team, or has sufficient League Manager access.
- Valid `espn_s2` and `SWID` cookies from that authorized account.

Second-manager access is sufficient and is safer than sharing another manager's login. It is scoped to the invited team and league, although it grants normal team-management capabilities—not logo-only permission. The League Manager must add the account specifically as a second manager of the existing team; a generic league invitation can create or assign a different team.

## Exact implementation steps

### 1. Identify the target

Read the IDs from the ESPN team URL:

```text
https://fantasy.espn.com/hockey/team?leagueId=1615029211&seasonId=2027&teamId=10
```

For this example:

```text
LEAGUE_ID=1615029211
SEASON_ID=2027
TEAM_ID=10
```

### 2. Prepare a 450 × 450 JPEG

For an already-square PNG on macOS:

```bash
LOGO_SOURCE='/absolute/path/to/source.png'
LOGO_OUTPUT='/absolute/path/to/team-logo.jpg'

sips -Z 450 \
  -s format jpeg \
  -s formatOptions 75 \
  "$LOGO_SOURCE" \
  --out "$LOGO_OUTPUT"

sips -g pixelWidth -g pixelHeight -g format "$LOGO_OUTPUT"
ls -lh "$LOGO_OUTPUT"
```

Expected output: 450 × 450, JPEG, under 500 KB. Crop non-square artwork before conversion; `sips -Z` preserves the existing aspect ratio rather than cropping.

### 3. Add and publish the asset

Use this repository layout:

```text
logos/<LEAGUE_ID>/team-<TEAM_ID>-<slug>.jpg
```

Example:

```text
logos/1615029211/team-10-zorros.jpg
```

Commit and push the image:

```bash
git add logos/1615029211/team-10-zorros.jpg
git commit -m 'Add team 10 logo'
git push origin main
```

Use the jsDelivr `@main` URL with ESPN:

```text
https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/<LEAGUE_ID>/team-<TEAM_ID>-<slug>.jpg
```

The `@main` form is the URL pattern verified in ESPN's live renderer. If replacing an existing file, allow jsDelivr time to refresh or use a new filename.

Verify the CDN asset before updating ESPN:

```bash
LOGO_URL='https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/1615029211/team-10-zorros.jpg'
curl --fail --silent --show-error --location --head "$LOGO_URL"
```

The response must be HTTP 200 with `Content-Type: image/jpeg`.

### 4. Load credentials safely

Obtain `espn_s2` and `SWID` from the logged-in ESPN browser session. Treat both as secrets. Do not put them in this public repository, screenshots, chat messages, command examples, or committed files.

For a temporary zsh session, enter them without placing the values in the command itself:

```zsh
read -s 'ESPN_S2?ESPN_S2: '
echo
read 'SWID?SWID: '

export ESPN_S2 SWID
export LEAGUE_ID=1615029211
export SEASON_ID=2027
export TEAM_ID=10
export LOGO_URL='https://cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/logos/1615029211/team-10-zorros.jpg'
```

### 5. Write the team logo

Fantasy Hockey uses game code `fhl` and segment `0`:

```text
POST https://lm-api-writes.fantasy.espn.com/apis/v3/games/fhl/seasons/<SEASON_ID>/segments/0/leagues/<LEAGUE_ID>/teams/<TEAM_ID>
```

Run:

```bash
set -euo pipefail

TEAM_WRITE_URL="https://lm-api-writes.fantasy.espn.com/apis/v3/games/fhl/seasons/${SEASON_ID}/segments/0/leagues/${LEAGUE_ID}/teams/${TEAM_ID}"
TEAM_PAGE_URL="https://fantasy.espn.com/hockey/team?leagueId=${LEAGUE_ID}&seasonId=${SEASON_ID}&teamId=${TEAM_ID}"
ESPN_COOKIE="espn_s2=${ESPN_S2}; SWID=${SWID}"
LOGO_PAYLOAD=$(jq -cn --arg logo "$LOGO_URL" '{logo:$logo,logoType:"CUSTOM"}')

curl --fail-with-body --silent --show-error \
  --request POST "$TEAM_WRITE_URL" \
  --header "Cookie: $ESPN_COOKIE" \
  --header 'X-Fantasy-Platform: espn-fantasy-web' \
  --header 'X-Fantasy-Source: kona' \
  --header 'Origin: https://fantasy.espn.com' \
  --header "Referer: $TEAM_PAGE_URL" \
  --header 'Accept: application/json' \
  --header 'Content-Type: application/json' \
  --data "$LOGO_PAYLOAD"
```

Do not include the team name unless it also needs to change. A successful response reports the logo as `CUSTOM_VALID`.

### 6. Verify the league record

```bash
TEAM_READ_URL="https://lm-api-reads.fantasy.espn.com/apis/v3/games/fhl/seasons/${SEASON_ID}/segments/0/leagues/${LEAGUE_ID}?view=mTeam"

curl --fail --silent --show-error "$TEAM_READ_URL" \
  --header "Cookie: $ESPN_COOKIE" \
  --header 'X-Fantasy-Platform: espn-fantasy-web' \
  --header 'X-Fantasy-Source: kona' \
  --header 'Accept: application/json' \
  --header 'Cache-Control: no-cache' \
  | jq --argjson teamId "$TEAM_ID" \
      '.teams[] | select(.id == $teamId) | {id,name,logoType,logo}'
```

Expected fields:

```json
{
  "logoType": "CUSTOM_VALID",
  "logo": "https://cdn.jsdelivr.net/gh/...@main/...jpg"
}
```

### 7. Synchronize the delegated preference

In the ESPN Fantasy mobile app, using an owner or second-manager account:

1. Open the target team.
2. Make a real team-name change.
3. Save it.
4. If the original name should remain, restore it and save again.
5. Allow ESPN time to propagate the change.

For Fantasy Hockey, the delegated preference ID follows this pattern:

```text
<TEAM_ID>:<LEAGUE_ID>:4:<SEASON_ID>
```

The `4` is ESPN's game ID for Fantasy Hockey. The mobile save is the safe way to make ESPN refresh this delegated record; do not POST logo changes to the fan-preference API.

### 8. Reload and confirm

After the API write and mobile save:

1. Reload the page with `Cmd+R`.
2. If necessary, reload from origin with `Cmd+Option+R`.
3. Close and reopen the ESPN tab or all Private Browsing windows.
4. Allow time for ESPN and jsDelivr caches to propagate.

The page may briefly display the custom logo before showing the shield if Safari has stale client state. When every ESPN record is already `CUSTOM_VALID`, a normal reload has been observed to load and retain the custom image.

## Troubleshooting

### The API returns 401 or 403

- Refresh `ESPN_S2` and `SWID` from the logged-in browser.
- Confirm the authenticated account owns the target team or is listed as its second manager.
- Confirm `LEAGUE_ID`, `SEASON_ID`, and `TEAM_ID` came from the same team URL.

### The write returns 400

- Confirm the body is exactly `{logo: <public URL>, logoType: "CUSTOM"}`.
- Confirm `Content-Type: application/json` is present.
- Do not send read-only team fields back to the endpoint.

### ESPN reports `CUSTOM_VALID`, but the page shows the shield

- Open the CDN URL directly and confirm it returns the intended JPEG.
- Confirm both the league API and mobile-synchronized preference contain the custom URL.
- Reload the page after caches have had time to propagate.
- In Safari, open Web Inspector → Network, reload, and filter for the filename or `jsdelivr`.
- The image request must finish with HTTP 200 and `Content-Type: image/jpeg`.

### The custom logo appears briefly, then changes to the shield

This indicates client-side fallback or stale hydration rather than a failed ESPN write. Verify the network request, then reload. In live testing, opening the Network panel and performing a normal reload caused the custom logo to remain stable once both ESPN records already contained the `@main` URL.

### A mobile save restores an old name or shield

The delegated fan preference was stale when the mobile save ran. Reapply the custom URL through the team-write endpoint, then perform the mobile name-save again. Do not try to repair this by writing directly to the fan-preference endpoint.

## Technical reference

| Purpose | Endpoint or value |
| --- | --- |
| Hockey game code | `fhl` |
| Hockey fan-preference game ID | `4` |
| Season segment | `0` |
| Team write | `lm-api-writes.fantasy.espn.com/apis/v3/games/fhl/seasons/.../teams/...` |
| Team read | `lm-api-reads.fantasy.espn.com/apis/v3/games/fhl/seasons/.../leagues/...?view=mTeam` |
| Web platform header | `X-Fantasy-Platform: espn-fantasy-web` |
| Web source header | `X-Fantasy-Source: kona` |
| Requested logo type | `CUSTOM` |
| Accepted logo type | `CUSTOM_VALID` |
| Recommended asset | 450 × 450 JPEG, quality 75, under 500 KB |
| Recommended CDN path | `cdn.jsdelivr.net/gh/anton-g-kulikov/espn-fantasy-logos@main/...` |
