plexStack — Plex + Arrs + Transmission + Prowlarr (via torproxy) + Portal

Operator notes. Keep internals on plexStackNetwork, expose only the ports you need.

What’s in the stack
	•	Plex, Transmission, Sonarr, Radarr, Prowlarr (behind torproxy), Overseerr, Portal (nginx)

Host paths (expected)

/Misc/plexStack/
  media/{Movies,Shows}
  transfer/{incomplete,complete,watch}
  apps/
    plex/config/  transmission/{config,watch}
    sonarr/config/ radarr/config/ prowlarr/config/
    overseerr/config/ portal/www/

External ports (host → container)
	•	Portal 61000→80, Overseerr 61055→5055, Sonarr 61089→8989, Radarr 61078→7878,
Transmission UI 61091→9091, Transmission peers 61413 TCP/UDP→61413,
Prowlarr UI via torproxy 61096→9696, Plex 32400→32400.

Edit before up -d
	•	Change /Misc/plexStack:/plexStack if needed; set PUID/PGID/TZ.
	•	Plex ADVERTISE_IP: http://<LAN-IP>:32400/.
	•	Transmission: in apps/transmission/config/settings.json set "peer-port": 61413, "peer-port-random-on-start": false; forward 61413 TCP+UDP on your router.

Bring-up

docker compose pull
docker compose up -d

Initial setup (required)

1) Transmission
	•	Paths:
	•	Download /plexStack/transfer/complete
	•	Incomplete /plexStack/transfer/incomplete
	•	Watch /plexStack/transfer/watch
	•	Optional: ratio stop (e.g., 0.5) and idle stop (e.g., 60m).

2) Prowlarr (http://:61096)
	•	(Optional) Proxy: HTTP 127.0.0.1:8118 via torproxy.
	•	Add a public indexer: Indexers → + → Public site (pick one that works for you), test & save.
	•	Connect apps: Apps → +
	•	Sonarr: URL http://sonarr:8989 (API key from Sonarr → Settings → General).
	•	Radarr: URL http://radarr:7878 (API key from Radarr).
	•	Enable Sync so indexers push into Arr.

3) Sonarr (http://:61089)
	•	Root folder: /plexStack/media/Shows.
	•	Download client: Transmission → URL http://transmission:9091 (use RPC user/pass).
	•	Media Management: Use Hardlinks instead of Copy = On.
	•	Profiles: set your preferred quality (e.g., 2160p only).

4) Radarr (http://:61078)
	•	Root folder: /plexStack/media/Movies.
	•	Download client: Transmission → http://transmission:9091.
	•	Profiles: 4K-only if desired; set cutoff.

5) Overseerr (http://:61055)
	•	Sign in with Plex.
	•	Applications: add Sonarr and Radarr:
	•	Sonarr URL http://sonarr:8989, Radarr URL http://radarr:7878.
	•	Pick root folders and profiles.
	•	(Optional) Lock down access (disable open registration, or use Plex-only auth).

6) Plex
	•	Libraries: add /plexStack/media/Movies and /plexStack/media/Shows.

Security options
	•	Bind admin UIs to loopback to keep them LAN-only:

sonarr: ports: ["127.0.0.1:61089:8989"]
radarr: ports: ["127.0.0.1:61078:7878"]
transmission: ports:
  - "127.0.0.1:61091:9091"
  - "61413:61413/tcp"
  - "61413:61413/udp"
portal: ports: ["127.0.0.1:61000:80"]
torproxy: ports: ["127.0.0.1:8118:8118","127.0.0.1:9050:9050"]


	•	Only forward externally: Plex 32400, Overseerr 61055, Transmission peers 61413 TCP+UDP.

Post-deploy TODO
	•	Confirm Transmission shows Port: 61413 (Open).
	•	Prowlarr indexer tests are green; Sync pushed to Sonarr/Radarr.
	•	Sonarr/Radarr Completed Download Handling: Enable = On; Remove = On (if you don’t want seeding after import).
	•	Drop a .torrent into /Misc/plexStack/transfer/watch → verify import → Plex sees it.

Quick checks

docker ps --format 'table {{.Names}}\t{{.Ports}}'
docker exec -it plexStack-transmission transmission-remote -si | egrep -i 'Port|UPnP|NAT-PMP'

Backups
	•	Save ./apps/*/config (Arr DBs, Plex metadata, Transmission settings). Media and downloads are outside.
