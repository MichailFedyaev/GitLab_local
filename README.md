# GitLab_local

Настройка собственного GitLab CI/CD сервера с помощью Docker Compose

# Введение
Сегодня я расскажу вам как развернуть собственный GitLab сервер со всеми необходимыми сервисами (dind, gitlab-runners, register-runner) с использованием Docker Compose.
Этот гайд поможет вам создать локальную среду для практики с CI/CD процессами. Мы пройдем через все этапы: от настройки контейнеров до регистрации раннеров и создания примера CI/CD пайплайна.

# Как запустить свой сервер GitLab в контейнере?

## 1. Подготовка файловой структуры:
```
mkdir gitlab-local
cd gitlab-local

# Создайте директории для хранения данных
mkdir -p ./data/docker/gitlab/{var/opt/gitlab,var/log/gitlab,etc/gitlab-runner,var/run/docker.sock}
mkdir -p ./config
touch ./config/config.toml
```
## 2. Запуск Docker Compose:

-Создайте файл ```docker-compose.yml``` в директории ```gitlab-local``` 
-Скопируйте содержимое файла к себе в .yaml
 [Ссылка на docker-compose.yml](https://github.com/MichailFedyaev/GitLab_local/blob/main/docker-compose.yml)
 ```nano docker-compose.yml```  
-Теперь запустите контейнеры:  
 ```docker-compose up -d```  
-После чего проверьте логи контейнера dind  
 если видете такую ошибку  
 ```
 failed to start daemon: error initializing graphdriver:  
 error changing permissions on file for metacopy check: permission denied: overlay2
 ```  
 то советую на время настройки поменять драйвер контейнер dind с ```"--storage-driver=overlay2",``` на ```"--storage-driver=vfs",```  
 Это связано с тем, что директории создаются на файловой системе Windows (NTFS), которая не поддерживает UNIX права.  
 Временно решение:
    Используйте драйвер ```vfs``` вместо ```overlay2```:
    ```
    command: [
    "--host", "tcp://0.0.0.0:2375",
    "--storage-driver=vfs",
    "--tls=false"
    ]
    ```
```vfs``` менее производительный чем ```overlay2```, но и менее требовательный  
Поэтому на стадии настройки лучше поставить именно его.  
 Решение:  
    Переместите данные в файловую систему Linux внутри WSL.  
## 3. Доступ к веб-интерфейсу GitLab:

- URL: ```http://localhost:8000/```
- Логин: ```root```
- Пароль: ```CHANGEME123```

## 4. Настройка SSH-ключа:

- Перейдите в настройки пользователя GitLab  
   URL: ```http://localhost:8000/-/user_settings/ssh_keys```  
- Добавьте новый SSH-ключ  

# Регистрация и настройка GitLab Runner для выполнения задач в Docker

## 1.Создайте и заполните файл .env 
```
nano .env
RUNNER_NAME=ИмяПроекта
REGISTRATION_TOKEN=ВашТокенРегистрации
```
## 2.Теперь нужно получить REGISTRATION_TOKEN:
- Создайте новый проект (```test```)  
- Зайдите в Settings → CI/CD → Runners (```http://localhost:8000/root/test/-/settings/ci_cd```)  
- Скопируйте Registration Token  
  ![image](https://github.com/user-attachments/assets/d7da7a74-0be7-49a9-a6d1-a8f56d5994fd)  
- Обновите файл .env с новым токеном и именем раннера (```test```).

## 3.Перезапустите сервис:  
```docker-compose restart register-runner```

# Пример настройки проекта с GitLab CI/CD  
## 1.Клонируйте проект который создали ранее (```test```)  
  ```git clone http://localhost:8000/root/test.git```  
  (или по SSH ```git clone ssh://git@localhost:8822/root/test.git```)  
## 2.Создаем ветку и файлы  
  ```git checkout -b test_ci```  
## 3.Создаем файл main.py  
  ```
  def main():
    """
    This function prints "Hello World" to the console.
    """
    print('Hello World')

  main()
  ```
## 4.Создаём файл .gitlab-ci.yml  
- Скопируйте содержимое файла к себе в .yaml  
  [Ссылка на docker-compose.yml](https://github.com/MichailFedyaev/GitLab_local/blob/main/docker-compose.yml)
  ```nano .gitlab-ci.yml.yml```  
## 5.Фиксируем изменения ```git add .```
## 6.Комитим изменения ```git commit -m "feat: #12345 Добавлен CI пайплайн"```
## 7.Отправляем изменения ```git push -u origin test_ci```
## 8.После чего создаем Merge Request
  Открываем созданный проект ```test```
  Нажмите "Create merge request" для ветки ```test_ci```
## 9.Проверьте выполнение пайплайна на вкладке "Сборочная линия"
  В проекте: ```CI/CD``` → ```Pipelines```
  Должны отображаться два задания: ```commit-name-test``` и ```pylint```

  ## Этот гайд поможет вам настроить локальную среду GitLab CI/CD для изучения и экспериментов. Помните, что для продакшен-среды потребуются дополнительные настройки безопасности и производительности.
