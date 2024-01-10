---
tags:
  - generative ai
  - ollama
  - mistral
---

# How to install Ollama with a ChatGPT-like web UI

_2024-01-10_

I am assuming that this is being installed on Ubuntu Server. Instructions compiled from:

- https://www.zdnet.com/article/docker-101-how-to-install-docker-on-ubuntu-server-22-04/
- https://ollama.ai/download/linux
- https://github.com/jmorganca/ollama/blob/main/docs/faq.md
- https://github.com/ollama-webui/ollama-webui

First, install Ubuntu Server, ensuring that you have a modern CPU with AVX2 support as we're going to be using the CPU to run the model, rather than a GPU. You'll also need at least 8GB RAM for the model to run because we're using [Mistral 7B 0.2](https://ollama.ai/library/mistral), which requires roughly 7GB RAM to run.

## Install Ollama

Next we can install Ollama itself. Run the following command to download and install Ollama:

```bash
curl https://ollama.ai/install.sh | sh
```

Next, really insecurely open up Ollama to your entire network, or more securely to just specific IP addresses or domains. For this example, I'm going to expose it to the entire network by running these commands:

```bash
mkdir -p /etc/systemd/system/ollama.service.d
echo '[Service]' >>/etc/systemd/system/ollama.service.d/environment.conf
```

N.B. Change this environment variable if you want to make it more secure than `0.0.0.0`.

```bash
echo 'Environment="OLLAMA_HOST=0.0.0.0:11434"' >>/etc/systemd/system/ollama.service.d/environment.conf
```

```bash
systemctl daemon-reload
systemctl restart ollama
```

## Install the Mistral 7B 0.2 model

Now we can install the Mistral 7B 0.2 model. Run the following command to download and install the model:

```bash
ollama run mistral
```

Once it is downloaded you can press ctrl+d to exit.

## Install Docker

This is a prerequisite for the web ui. Run the following commands to install Docker:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bash
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y
```

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

Optionally you can add your user to the docker group so you can run `docker` commands without `sudo`. If you do this, you'll need to log out and log back in (or just reboot) for this change to take effect.

```bash
sudo usermod -aG docker $USER
```

## Install the Ollama web UI

Run this command to create and start a new docker container running the web ui on port 3000:

```bash
docker build -t ollama-webui .
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway \
-v ollama-webui:/app/backend/data --name ollama-webui --restart always \
ollama-webui
```

N.B. The documentation for this project on GitHub includes examples for if you have Ollama running on a different machine.

## Access the web UI

Finally you can visit your Ubuntu machine's IP address with port 3000 and create a new admin account. You can then log in and start using the web UI, selecting your model(s) and entering a prompt to generate text from.

## Bonus model

If you want to use a model specifically as a coding assistant then the best currently available is Mixtral...which needs at least 48GB RAM to run. Personally, I'm using the 6.7 billion parameter version of the deepseek-coder model. If you want to install it you can run: 

```bash
ollama run deepseek-coder:6.7b
```

It is then selectable from the dropdown at the top of the web UI when you start a conversation.
