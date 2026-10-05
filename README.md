## Measurement Data For "What Location Data Does An iPhone Send To Apple?"

Paper: D.J. Leith, "Decoding What iOS Location Services Send to Apple: An LLM Is an Unreliable Analyst but a Useful Toolmaker", Trinity College Dublin. [link to be added]

This repository holds the network connections that an iPhone 8 (iOS 16.7.11) makes to Apple's location services, decoded to readable text. We recorded them in three experiments (location off, location on, and on car journeys) by decrypting the handset's HTTPS traffic with `mitmdump`, and we decoded the protobuf messages with [ios_decoder](https://github.com/doug-leith/ios-protobuf-decoding) using the message definitions recovered by Extractor, the tool described in the paper. The paper analyses these data. Here you can read the connections themselves, for example to check a claim in the paper or to look at behaviour that the paper does not discuss.

The data are redacted before publication (see *Redaction* below). 

### Files

Everything is in `traces/<experiment>/`, one gzipped text file per process per experiment. Each file contains every HTTPS connection that the process made in that experiment, with request and response bodies decoded.

| File | What it contains | Connections in experiment 1 / 2 / 3 |
|---|---|---|
| `geod.txt.gz` | `geod`, the geo services daemon: configuration, map tile and resource downloads, A/B experiment assignment, usage telemetry, Wi-Fi quality lookups, reverse geocoding for other processes, and the real-time traffic probe (`rtloc`). | 299 / 300 / 135 |
| `geod-20H360.txt.gz` | `geod`'s tile and asset downloads. These requests identify themselves by the iOS build number (`20H360`) instead of `geod/1`. | 6 / 38 / 164 |
| `mobileasset-geod.txt.gz` | The requests that `geod`'s MobileAsset client makes to `mesu.apple.com` and `updates.cdn-apple.com`. | 3 / 7 / 2 |
| `locationd.txt.gz` | `locationd`, the location daemon: Wi-Fi and cell lookups (`wloc`), tile downloads, GNSS assistance data, and the uploads of collected sensor data (`/hcy/pbcwloc`, `/pds/pd`, `/hvr/trc`, `/psr`). | 7 / 393 / 158 |
| `photoanalysisd.txt.gz` | The photo analysis daemon, which sends reverse geocoding and point-of-interest requests to Apple's geo servers directly. Only its connections to those servers are kept. | 0 / 4 / 0 |
| `mapspushd.txt.gz` | The Maps daemon, which fetches announcements directly. Only its connections to Apple's geo servers are kept. | 4 / 8 / 2 |
| `client-<name>.txt.gz` | The connections that `geod` makes on behalf of one of its clients: `analyticsd`, `corelocation-prompt-plugins`, `coreroutine-helper`, `kbd`, `networkserviceproxy`, `threebars-wifi-quality` (the Wi-Fi quality service) and `weather-widget`. These are copies of connections that are also in `geod.txt.gz`, sorted by client. | 98 / 94 / 8 in total |

In this table a connection means one HTTPS request and its response.

### The experiments

All times are UTC. The windows are those used to cut each experiment out of the raw recordings.

| Folder | When | Conditions |
|---|---|---|
| `expt1_static_location_off` | 17 Feb 09:00 to 18 Feb 12:10 2026 | Factory reset, onboarding with Location Services off and no Apple Account. The handset is stationary on the test Wi-Fi network and idle. |
| `expt2_static_location_on` | 18 Feb 12:19 to 6 Mar 18:00 2026 | The same handset with Location Services on, still stationary on the test Wi-Fi network. The user uses the App Store and signs in to an Apple Account (some sign-ins are on a different Wi-Fi network that we did not capture). |
| `expt3_drive` | 4 Jun 05:00 to 5 Jun 21:00 2026 | The handset on mobile data with all traffic sent through a VPN, carried on four car journeys (morning and evening on two days). Apple Maps is opened briefly on 4 June to check that the reported location is correct. |

The handset was configured as a privacy-conscious but busy user would configure it: it declines every option that shares data with Apple and otherwise keeps the default settings. The paper describes the test protocol and lists the settings chosen during onboarding.

### Reading a trace

Each connection is a block of text that starts with a blank line and a `timestamp` line, followed by the request line and the decoded bodies, followed by marker lines that start with `+++`:

```
timestamp 1771326379.9549763 (2026-02-17 11:06:19.954976+00:00 UTC)
POST https://gsp-ssl.ls.apple.com/ab.arpc
Request body (decoded):
apple arpc header: version=1, locale=en-IE_IE, app=com.apple.geod, osVersion=16.7.11.20H360, requestType=1016, length=83
guid: "53101232-<redacted>"
clientMetadata {
  deviceCountryCode: "IE"
  deviceDisplayLanguage: "en-IE"
  [...]
}
Response body (decoded):
apple header: version=1, responseType=1016, payloadLength=80
requestGuid: "53101232-<redacted>"
refreshIntervalSeconds: 14400
parsecClientMetadata {
  clientDatasetMetadata {
    bucketId: 7428
  }
}
[...]
branchExpirationTtlHours: 720
+++REQUEST https://gsp-ssl.ls.apple.com/ab.arpc 233 1771326379.9549763
+++REQHEADERS https://gsp-ssl.ls.apple.com/ab.arpc € User-Agent='geod/1 CFNetwork/1410.1 Darwin/22.6.0' € accept=*/*~content-type=application/octet-stream~accept-encoding=gzip, deflate, br~[...]
+++RESPONSE 200 - 105
+++RESPHEADERS https://gsp-ssl.ls.apple.com/ab.arpc € - € server=Apple~date=Tue, 17 Feb 2026 11:06:19 GMT~[...]
```

(The `[...]` marks parts that we have left out of this example.)

* **Times.** The `timestamp` line gives the time of the request as a Unix time, followed by the same time in UTC. 
* **Protobuf bodies** are printed with field names. Many Apple requests are wrapped in an ARPC envelope, whose header is printed on the first line of the body. A field that the handset's message definition lacks (the server sends fields that this iOS version does not declare) is printed by number, and a `// note:` line marks it. Two kinds of body are printed field by field by number because no definition matches them: the response of `cl4.apple.com/2/m2/a.1//availability.pb.gz` and the network-selection tiles of `gspe79-ssl.ls.apple.com/72/v2/72/<key>`.
* **Other bodies** (property lists, JSON, map tiles and so on) are decoded by format. A body that we do not decode is shown as its first 64 bytes. Map tile requests are followed by a line such as `+++XYZ <url> (x,y,z)` that gives the tile coordinates, and the tiles of `locationd` by `+++LATLON_BOUNDBOX <name> (lat0,lat1,lon0,lon1)`. These are the lines that we use to draw the tile figures in the paper.
* **Marker lines.** `+++REQUEST` gives the URL, the size of the request body and the request time. `+++REQHEADERS` and `+++RESPHEADERS` list the HTTP headers as `name=value` pairs separated by `~`. `+++RESPONSE` gives the HTTP status, the content type and the size of the response body. Credentials in headers are replaced by `<redacted>`.
* **Size cap.** The decoded body of one connection is cut at 250,000 bytes, and a line starting `// [archive: decoded output truncated` marks the cut. This only affects large downloads such as tiles and `geo_manifest`. Every marker line is kept.

To find a connection, search for its URL. For example `gzip -dc traces/expt3_drive/locationd.txt.gz | grep -A12 'POST https://gsp10-ssl.apple.com/pds/pd'` prints the start of the GPS upload that the paper discusses.

### Redaction

The traces contain the handset's own location (hundreds of times, including its home), its hardware identifiers and the Wi-Fi MAC addresses of neighbouring access points, so we redact them before publication.

| What | How it is redacted |
|---|---|
| Latitude and longitude (named fields, and unnamed 64-bit values in the coordinate range of Ireland) | Truncated to two decimal places, about 1 km: `53.31<redacted>` |
| MAC addresses | The lower three bytes are masked: `00:a3:88:xx:xx:xx` |
| Handset identifiers (IMEI, ICCID, MEID, serial number, SEID, ECID), e-mail addresses, phone numbers | Replaced by `<imei>`, `<email>` and so on |
| Quoted UUIDs in message fields (trip identifiers, session GUIDs) | Only the first eight hex digits are kept, so that records that share an identifier can still be seen to share it |
| Cookies, authorisation and device-identifier headers | `<redacted>` |
| Street addresses and place names in reverse geocoding responses | Replaced by `<redacted string, N chars>` |

We do not redact cell tower identifiers, accuracies, timestamps, the tile coordinates in URLs, or the contents that appear in responses (the positions of access points and cell towers in the tile data). We also keep the `accessKey` URL parameters and `maps-auth-token` headers of the map downloads, because the paper shows that they are not specific to the handset. 

### What is not included

* The raw recordings (about 4 GB), which contain the unredacted data, cookies and account tokens.
* The traffic of other processes (the App Store, iCloud etc). It is outside the scope of the paper and it carries account tokens. A few of the `geod` connections were made for apps that the experimenter opened (Apple Maps on 4 June), and they name the app.
* The decoder and the tool that recovers the message definitions. They are in the [ios-protobuf-decoding](https://github.com/doug-leith/ios-protobuf-decoding) repository.


### How the data were collected

We used an iPhone 8 running iOS 16.7.11 (build 20H360), jailbroken with [palera1n](https://github.com/palera1n/palera1n) 2.2.1 and with the [Ellekit](https://github.com/tealbathingsuit/ellekit) function hooker. A [modified version of SSL KillSwitch](https://github.com/doug-leith/poor-privacy-practices-of-the-apple-app-store/tree/main/ssl-kill-switch3-DL) disables the certificate pinning that system processes use, which allows `mitmdump` 13.0.0.dev to decrypt their connections. In experiments 1 and 2 the handset connected through a Wi-Fi access point that we control, which redirects all HTTP and HTTPS traffic to `mitmdump` so that the proxying is transparent to the handset. In experiment 3 the handset used mobile data and sent all traffic through a VPN server that runs `mitmdump` in the same way. The paper gives the details.

### Checking the files

`SHA256SUMS` lists the SHA-256 checksum of every trace file. To check them, run `sha256sum -c SHA256SUMS`.
