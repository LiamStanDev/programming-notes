> 以下建議都使用 docker 安裝，這樣 WebUI + Ollama 才能夠通過 docker internal network 進行交互。


### 安裝 ollama
reference: 
1. [ollama](https://github.com/ollama/ollama/blob/main/docs/docker.md)
2. [nvidia docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installation)
```shell
# install nvidia docker
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
sudo dnf config-manager --set-enabled nvidia-container-toolkit-experimental
sudo dnf install -y nvidia-container-toolkit

# configure docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# run ollama docker
docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```


### WebUI
reference: [open-webui](https://github.com/open-webui/open-webui)
```shell
docker run -d -p 3000:8080 --gpus all --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:cuda
```