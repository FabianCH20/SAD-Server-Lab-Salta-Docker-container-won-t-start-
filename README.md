<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:000000,100:00FF41&height=180&section=header&text=SadServers%20//%20Salta&fontSize=42&fontColor=00FF41&animation=fadeIn&fontAlignY=38&desc=Docker%20container%20won't%20start&descAlignY=58&descColor=00FF41" />

<img src="https://readme-typing-svg.demolab.com/?font=Fira+Code&size=18&duration=2500&pause=800&color=00FF41&center=true&vCenter=true&width=600&lines=root%40sadservers%3A~%23+diagnosing+container...;root%40sadservers%3A~%23+port+8888+refused...;root%40sadservers%3A~%23+access+granted+%E2%9C%93" />

<br/>

![Level](https://img.shields.io/badge/LEVEL-MEDIUM-00FF41?style=for-the-badge&labelColor=000000)
![Platform](https://img.shields.io/badge/PLATFORM-SadServers-00FF41?style=for-the-badge&labelColor=000000)
![Stack](https://img.shields.io/badge/STACK-Docker%20%7C%20Node.js-00FF41?style=for-the-badge&labelColor=000000)
![Status](https://img.shields.io/badge/STATUS-SOLVED-00FF41?style=for-the-badge&labelColor=000000)

</div>

<br/>

```bash
krikox@matrix:~$ cat mission_briefing.txt
```

> There's a "dockerized" Node.js web application in the `/home/admin/app` directory.
> Create a Docker container so you get a web app on port **:8888** and can curl to it.
> For the solution to be valid, there should be only **one** running Docker container.
>
> **Test:** `curl localhost:8888` returns `Hello World!` from a running container.

<br/>

## 📋 `./01_recon.sh`

```bash
krikox@matrix:~$ cd app/ && ls
Dockerfile  package.json  server.js

krikox@matrix:~$ cat package.json
# → confirmed the Node.js project's start script and dependencies

krikox@matrix:~$ cat Dockerfile
# → reviewed the EXPOSE and CMD directives of the image
```

<details>
<summary><b>🔎 Initial findings</b></summary>
<br/>

| File | Status |
|---|---|
| `Dockerfile` | present, with **2 misconfigurations** |
| `package.json` | correct |
| `server.js` | correct — the app's actual entrypoint |

</details>

<br/>

## 🛠️ `./02_fix_dockerfile.sh`

Two errors were found in the original `Dockerfile`:

```diff
- EXPOSE 8880
+ EXPOSE 8888

- CMD ["node", "serve.js"]
+ CMD ["node", "server.js"]
```

```bash
krikox@matrix:~$ nano Dockerfile
# → port corrected to 8888
# → entrypoint corrected to server.js (matches the actual file, case-sensitive)
```

<br/>

## 🐳 `./03_build_and_run.sh`

```bash
krikox@matrix:~$ docker build . -t app
krikox@matrix:~$ docker run -d app
krikox@matrix:~$ docker ps
# → "app" image running correctly
```

<br/>

## 🚧 `./04_troubleshooting.sh`

Running `curl localhost:8888` still didn't respond. Port investigation:

```bash
krikox@matrix:~$ docker ps -a
krikox@matrix:~$ ss -tulpn | grep 8888
# → port 8888 already taken by an nginx container + the host's nginx service
```

```bash
krikox@matrix:~$ docker rm <nginx_container_id>
krikox@matrix:~$ systemctl stop nginx
krikox@matrix:~$ systemctl status nginx
# → nginx service stopped, port freed

krikox@matrix:~$ docker restart <app_container_id>
krikox@matrix:~$ curl localhost:8888
curl: (7) Failed to connect to localhost port 8888: Connection refused
```

<details>
<summary><b>🧠 Root cause</b></summary>
<br/>

`EXPOSE` in the Dockerfile only **documents** the port the container uses — it does
**not** publish it to the host. Without the `-p host:container` flag on `docker run`,
the port never gets mapped, no matter what the internal process is listening on.

Reference: [docker port not being exposed — Stack Overflow](https://stackoverflow.com/questions/63229280/docker-port-not-being-exposed)

</details>

<br/>

## ✅ `./05_final_fix.sh`

```bash
krikox@matrix:~$ docker run -d -p 8888:8888 app
krikox@matrix:~$ curl localhost:8888
Hello World!
```

<div align="center">

![Result](https://img.shields.io/badge/curl%20localhost%3A8888-Hello%20World!-00FF41?style=for-the-badge&labelColor=000000)

</div>

<br/>

## 💡 Lessons learned

```bash
krikox@matrix:~$ cat lessons_learned.log
```

- `EXPOSE` in a Dockerfile is documentation only; actual port mapping requires `-p host:container` on `docker run`.
- Before assuming the image is the problem, check for port conflicts with `ss -tulpn` or `netstat -tulpn` — another container or a host service may already be bound to it.
- File names in `CMD`/`ENTRYPOINT` are case-sensitive on Linux; they must match the real file exactly (`server.js` ≠ `serve.js`).
- The "only one running container" requirement means cleaning up duplicate or conflicting containers (`docker rm`) before validating the solution.

<br/>

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:00FF41,100:000000&height=100&section=footer" />

</div>
