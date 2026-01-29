# Help chat 504 on EC2

If you get **504 Gateway Timeout** when using the Help chat (Progento) from Cogento on EC2, check the following.

## 1. PROGENTO_API_URL is reachable from Cogento API

Cogento API runs **inside Docker**. From inside the container, `localhost` is the container itself, not the host. So Progento on the same EC2 host must be reached via the host's IP from the container.

In your `.env` (or environment for the Cogento API):

```bash
# Same EC2 host: use Docker bridge IP (Linux) so the Cogento API container can reach Progento on the host
PROGENTO_API_URL=http://172.17.0.1:8001
```

If Progento runs in Docker on the same host but in another compose project, use that container's IP or put both on a shared network and use the Progento service name and port.

Verify from the Cogento API container:

```bash
docker exec cogento-api python -c "
import urllib.request
try:
    urllib.request.urlopen('http://172.17.0.1:8001/api/health', timeout=5)
    print('Progento reachable')
except Exception as e:
    print('Progento not reachable:', e)
"
```

## 2. Reverse proxy timeout (nginx, ALB, etc.)

Help chat calls Progento's LLM; a single query can take **30–120 seconds** on CPU. If a reverse proxy in front of Cogento API has a short timeout (e.g. 60s), it will return **504** before the API responds.

- **nginx**: Increase timeouts for the API location, e.g.:
  ```nginx
  location /api/ {
      proxy_pass http://cogento-api:8000;
      proxy_connect_timeout 300s;
      proxy_send_timeout 300s;
      proxy_read_timeout 300s;
  }
  ```
- **AWS ALB**: In the Target Group or ALB settings, set **Idle timeout** to 300 seconds (default is 60).

## 3. Progento and Ollama

Ensure Progento and Ollama are running and the model set in `PROGENTO_HELP_CHAT_MODEL` (default `phi`) is installed in Ollama. Slow or missing models can cause timeouts or errors.
