# vMix → Staylive mid-roll ads

vMix scripts (VB.NET) that trigger mid-roll ad breaks on a Staylive
livestream through the Staylive messages API
(see [API MIDROLLS.md](API%20MIDROLLS.md)).

**PlayAd is the only script you need.** CancelAd is optional. Add it only if
you want a button that stops an ad early or cancels one sent by mistake.
Without it, every ad plays to the end.

| File | Purpose |
| --- | --- |
| `staylive_play_ad.txt` | **Required.** vMix script **PlayAd** – sends `PLAY_AD` to every viewer and saves the returned `messageId`. |
| `staylive_cancel_ad.txt` | *Optional.* vMix script **CancelAd** – cancels the last ad sent on the current stream (`DELETE`). |
| `staylive.cfg` | Per-game settings (stream ID, token, ad tag URL). The only file you should need to edit on game day. |
| `API MIDROLLS.md` | *For reference only.* Staylive's API documentation. You don't need it to use the scripts. |

## One-time setup

### 1. Set the folder path in the scripts

The scripts have the folder `D:\script\midrolls\` written into their code:

| Path | What |
| --- | --- |
| `D:\script\midrolls\staylive.cfg` | Config (you create it from this repo) |
| `D:\script\midrolls\staylive_log.txt` | Log, written by the scripts |
| `D:\script\midrolls\staylive_lastad_<streamId>.txt` | Last `messageId` per stream, used by CancelAd |

Copy `staylive.cfg` into the folder you want to use. If it is not
`D:\script\midrolls\`, replace `D:\script\midrolls\` with your own folder on
**every** line below. Keep the trailing `\`. The folder must already exist,
because the scripts don't create it.

**`staylive_play_ad.txt`**

| Line | Current code |
| --- | --- |
| 3 | `' Reads streamId / token / adUrl from D:\script\midrolls\staylive.cfg` (comment only) |
| 8 | `Dim cfgFile As String = "D:\script\midrolls\staylive.cfg"` |
| 9 | `Dim logFile As String = "D:\script\midrolls\staylive_log.txt"` |
| 42 | `Dim stateFile As String = "D:\script\midrolls\staylive_lastad_" & streamId & ".txt"` |

**`staylive_cancel_ad.txt`** (only if you use CancelAd)

| Line | Current code |
| --- | --- |
| 3 | `' Reads streamId / token from D:\script\midrolls\staylive.cfg` (comment only) |
| 9 | `Dim cfgFile As String = "D:\script\midrolls\staylive.cfg"` |
| 10 | `Dim logFile As String = "D:\script\midrolls\staylive_log.txt"` |
| 34 | `Dim stateFile As String = "D:\script\midrolls\staylive_lastad_" & streamId & ".txt"` |

If you use CancelAd, both scripts must point at the same folder. CancelAd reads the
`staylive_lastad_…` file that PlayAd writes, so if the paths differ, cancel
will never find the ad.

Tip: in a text editor, find-and-replace `D:\script\midrolls\` with your
folder in each script you use.

### 2. Production delay (PlayAd only)

In `staylive_play_ad.txt`:

```vb
Dim producerDelayMs As Long = 0
```

- **On-site production** (you see the action live): leave at `0`.
- **Off-site production** (you are watching the encoded stream): set it to
  your end-to-end delay in milliseconds, so the ad lines up with what you saw
  when you pressed the button.

### 3. Add the scripts to vMix

1. In vMix, open **Settings → Scripting**.
2. Add a script named exactly `PlayAd` and paste the contents of
   `staylive_play_ad.txt`.
3. *(Optional)* Add a script named exactly `CancelAd` and paste the
   contents of `staylive_cancel_ad.txt`.
4. Bind each script to a shortcut, controller button or trigger with
   **Function = `ScriptStart`** and **Value = `PlayAd`** (and **`CancelAd`**
   if you added it).

If you edit a script file later, paste it into vMix again. vMix does not
reload it from disk.

## Before each game

Edit `staylive.cfg` (in the folder from step 1). The scripts read it on every
run, so you don't need to restart vMix.

| Key | What to put there |
| --- | --- |
| `streamId` | Numeric livestream ID from the Staylive stream URL. Changes every game. |
| `token` | The JWT/API token you get from Staylive. Ask your Staylive contact for it. |
| `adUrl` | VAST ad tag URL. The default is the Hockeyettan Google Ad Manager tag. For another league or customer, change at least `iu=` (ad unit) and `description_url=`. Leave `correlator=` empty, because PlayAd fills it in on every request. |

Keep the `key=value` format on one line per key, with no quotes.

## Checking that it works

Everything is logged to `staylive_log.txt`:

| Log line | Meaning |
| --- | --- |
| `OK stream=… {"message":{…"sentTo":N,"failed":M…}}` | Ad sent. `sentTo`/`failed` are the delivery counts. |
| `ABORT - config file missing` / `incomplete config` | Wrong path in the script, or an empty key in the cfg. |
| `ERR … 401` | Token missing, expired or invalid. |
| `ERR … 403` | Usually a **wrong `streamId`**, not a bad token. |
| `502 but ad WAS delivered - do NOT re-fire` | Staylive returned an error but viewers got the ad. |
| `502 and no record - safe to fire again` | Nothing went out, so pressing PlayAd again is safe. |
| `CANCEL skipped - no messageId` | No ad has been sent on this stream yet (or it was already cancelled). |
| `CANCEL ERR … 404` | The ad was already gone. The saved ID is cleared. |

## Good to know

- **Don't press PlayAd twice for the same break.** Viewers who got the first
  trigger will get a second ad. Use CancelAd and then PlayAd if you need to
  redo it.
- The ad length comes from the VAST creative. The stream resumes when the ad
  ends or when CancelAd runs.
- Viewers whose connection drops at the moment of the trigger miss that ad.
  They don't get it later.
- CancelAd also removes the ad from Staylive's broadcast history.
