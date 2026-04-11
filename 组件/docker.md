## 使docker使用Nvidia驱动
```bash
# 添加 NVIDIA 包仓库
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 更新并安装
sudo apt update
sudo apt install -y nvidia-container-toolkit

# 配置 Docker 使用 nvidia 运行时
sudo nvidia-ctk runtime configure --runtime=docker

# 重启 Docker
sudo systemctl restart docker
```

|                |     |     |
| -------------- | --- | --- |
| docker compose |     |     |
| docker network |     |     |
| docker inspect |     |     |

## docker compose

## docker network

## docker inspect

