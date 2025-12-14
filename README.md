# Установка minikube на Ubuntu

```
# Скачиваем и инсталлируем kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Скачиваем и инсталлируем minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb

# Добавляем себе права
sudo usermod -aG docker $USER && newgrp docker

# Отключаем своп
sudo swapoff -a

# Запускаем minikube в режиме однонодового кластера k8s
minikube start --vm-driver=docker
```

# Проверка minikube и k8s кластера

<img width="403" height="156" alt="image" src="https://github.com/user-attachments/assets/36ccd113-0b06-47a9-b970-fff86c42434c" />

<img width="614" height="139" alt="image" src="https://github.com/user-attachments/assets/7b989066-64ef-4dd4-bc2e-897ce4ddc0f8" />

# Создание структуры каталогов

